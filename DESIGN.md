# git-hosting-service 詳細設計書

- 文書状態: 初期実装前・設計確定作業中
- 最終更新日: 2026-07-28
- 対象リポジトリ: `TakashiSasaki/git-hosting-service`
- 実装状態: 未着手

本書は、これまでの設計協議で確定した事項を、初めてこのリポジトリを読む開発者およびコーディングエージェントが実装へ移れる形に整理したものである。未決定事項は確定事項と混在させず、末尾の「残るブロッキング項目」に分離する。

本文中の **MUST**、**MUST NOT**、**SHOULD**、**MAY** は、それぞれ必須、禁止、推奨、任意を表す。

---

## 1. 目的

本システムは、単一のLinuxホスト上で直接運用するセルフホスト型Gitホスティングサービスである。

主要機能は次のとおりである。

- Git Smart HTTPによるclone、fetch、pull、push
- Firebase AuthenticationによるWeb/API利用者の認証
- Personal Access Token（PAT）によるGit HTTP Basic認証
- Principal単位のRepository所有
- bare repositoryの作成、一覧、詳細取得、論理削除
- Repository状態と認証情報を管理するSQLiteデータベース
- 管理者CLIによるmigration、restore、purgeおよび障害復旧
- migration状態を未認証で確認できる最小限のWeb表示

本システムは、GitHub全体を再実装するものではない。初期実装では、Repositoryホスティング、認証、ライフサイクル管理、REST API、および後続設計で確定するContents APIを中心とする。

---

## 2. 設計原則

### 2.1 単一ホスト・直接インストール

- Dockerは使用しない。
- Nginx、Node.js、fcgiwrap、Git、SQLiteをホストOSへ直接導入する。
- 単一ホスト内のローカルファイルシステムを前提とする。

### 2.2 外部公開点の限定

- 外部へHTTP/HTTPSを公開するのはNginxだけである。
- Node.jsは`127.0.0.1:3000`でのみ待ち受ける。
- fcgiwrapはTCPを使用せず、systemd管理のUnix domain socketだけで待ち受ける。

### 2.3 認可ロジックのNode.js集約

- Git URLの解析、Repository解決、READ/WRITE分類、PAT検証、公開範囲判定はNode.jsプロセスへ集約する。
- Nginxは認可ルールを再実装せず、元リクエストをNode.jsへ渡し、検証済みのCGI値を受け取る。
- NginxとNode.jsで同一URLを別々の規則で解釈してはならない。

### 2.4 Gitデータ転送のNode.js非経由

- Git packデータはNode.jsを通さない。
- Nginxからfcgiwrapを経由し、`git-http-backend`へ渡す。
- Node.jsは認証・認可・メタデータ管理に責務を限定する。

### 2.5 データ喪失防止

- Repositoryの通常削除は物理削除ではなく論理削除とする。
- bare repositoryはquarantineへ移動し、管理者が明示的にpurgeするまで保持する。
- 不完全処理の痕跡はDB上の状態として残し、自動的に上書きしない。

### 2.6 Fail closed

- 内部認可APIが停止、タイムアウト、または不正応答した場合、Gitアクセスは拒否する。
- 未知のGitパス、service、methodの組み合わせは許可しない。

### 2.7 仕様確定と実装既定値の分離

- 公開API、認証境界、状態遷移、データ保持方針は仕様として固定する。
- ワーカー数やタイムアウトなど運用値は初期既定値を持つが、設定可能とする。

---

## 3. 初期実装の範囲

### 3.1 含むもの

- Firebase ID tokenを用いたREST API認証
- Principalの自動作成
- PATによるGit HTTP Basic認証
- Git Smart HTTP
- Repository管理REST API
- SQLite migration
- migration statusの公開
- Repositoryの論理削除
- 管理者CLIの基本構造

### 3.2 初期実装で含めないもの

- SSH Git transport
- Git protocolによるTCP daemon
- Dumb HTTP
- Git LFS
- Git archive over HTTP
- Repository rename
- ユーザーによるRepository restore
- ユーザー向けRepository purge
- Branch protection
- Force-push制限
- 組織、チーム、共同所有、共同編集者権限
- Fork、pull request、issue、release、Actions相当機能
- 複数ホスト構成
- 外部オブジェクトストレージ
- 動的な自動スケーリング

Contents API、PAT管理API、監査ログなどの範囲は末尾の未決定事項で扱う。

---

## 4. 用語

### Principal

システム内部の利用主体。人間向けusernameとは独立し、UUIDv4で識別する。

### Firebase UID

固定された単一Firebase project内でFirebase Authenticationが発行する利用者識別子。

### Repository ID

Repositoryメタデータを識別するUUIDv4。Principal IDとは独立して生成する。

### Repository name

同一Principal内で一意な、人間可読のRepository名。

### PAT

Git HTTP Basic認証のpasswordとして使用するPersonal Access Token。

### ready repository

Git Smart HTTPの配信対象となるbare repository。

### staging

Repository作成中の一時配置領域。

### quarantine

論理削除済みRepositoryのbare repositoryを保持する領域。

---

## 5. システム構成

```text
Internet / Git client / Browser
              |
              v
          Nginx :443
          /         \
         /           \
REST/Web             /git/*
   |                    |
   v                    +-- auth_request --> Node.js internal Git auth API
Node.js :3000                              |
   |                                       +-- approved CGI values
   v                                       |
SQLite                                     v
                                      fcgiwrap socket
                                            |
                                            v
                                    git-http-backend
                                            |
                                            v
                                  bare repositories
```

### 5.1 Nginx

Nginxは次を担当する。

- TLS終端
- REST/WebのNode.jsへのreverse proxy
- Gitリクエストへの`auth_request`
- Git POST bodyの受信・バッファリング
- FastCGI経由の`git-http-backend`呼び出し
- migration status JSONの限定公開

NginxはRepositoryの所有者、visibility、READ/WRITEを判断しない。

### 5.2 Node.js

単一のNode.jsプロセスが次を担当する。

- 外部REST API
- Firebase ID token検証
- Principal解決・初回作成
- PATの検証
- Nginx向け内部Git認可API
- Git URLの厳密な解析
- Repository管理
- SQLiteアクセス

Node.jsはGit packのstreamingを担当しない。

### 5.3 fcgiwrap / git-http-backend

- fcgiwrapはsystemd socket activationで起動する。
- ワーカー初期値は2である。
- READとWRITEは同じワーカープールを共有する。
- `git-http-backend`はNginxから渡された検証済みCGI変数に基づいてGit Smart HTTPを処理する。

---

## 6. 配置とディレクトリ構造

### 6.1 永続データ

```text
/srv/custom-git-host/
├── repositories/
│   └── {principalId}/
│       └── {repositoryName}.git/
├── staging/
│   └── {repositoryId}.git/
└── quarantine/
    └── {repositoryId}.git/
```

- `repositories`、`staging`、`quarantine`は同一ファイルシステム上に置く。
- stagingからrepositories、repositoriesからquarantineへの移動はatomic renameを使用する。
- `GIT_PROJECT_ROOT`は`/srv/custom-git-host/repositories`に限定する。
- stagingとquarantineは`GIT_PROJECT_ROOT`の外でなければならない。

### 6.2 非公開runtime

```text
/run/custom-git-host/
└── git-http-backend.sock
```

- owner/group: `custom-git:custom-git`
- directory mode: `0700`
- FastCGI socketだけは後述の専用グループ権限を使用する。

### 6.3 公開可能なstatus runtime

```text
/run/custom-git-host-status/
└── migration-status.json
```

- directory owner/group: `custom-git:custom-git-status`
- directory mode: `0750`
- file owner/group: `custom-git:custom-git-status`
- file mode: `0640`
- NginxのOSユーザーだけを`custom-git-status`へ所属させる。
- Nginxは完全一致locationで`migration-status.json`だけを公開する。

### 6.4 Nginx request body一時領域

```text
/var/lib/custom-git-host/nginx-client-body/
```

このパスは専用ファイルシステムまたは容量制限付きマウントとし、OSルートおよびRepository保存領域から分離する。

---

## 7. OSユーザーとファイル権限

### 7.1 単一の専用実行ユーザー

初期実装では、Node.js系とGit実行系を分離せず、単一のOSユーザーを使用する。

```text
custom-git
```

`custom-git`が実行・所有するもの:

- Node.js
- migration CLI
- Repository管理CLI
- fcgiwrap
- `git-http-backend`
- SQLite database
- Firebase設定・秘密情報
- repositories
- staging
- quarantine

Node.jsとfcgiwrapは同じOSユーザーで動くが、systemd serviceは分離する。

将来のハードニングとして、アプリケーションユーザーとGit実行ユーザーを分離できるよう、責務境界はコード上で保つ。

### 7.2 基本権限

- 基本umask: `0077`
- 非公開directory: `0700`
- 非公開通常file: `0600`
- owner/group: `custom-git:custom-git`

`setgid`および`core.sharedRepository=group`は初期実装では使用しない。

### 7.3 FastCGI socket専用グループ

```text
custom-git-fcgi
```

- NginxのOSユーザーだけをこのグループへ所属させる。
- RepositoryやSQLiteの共有には使用しない。

socket:

```text
/run/custom-git-host/git-http-backend.sock
owner: custom-git
group: custom-git-fcgi
mode: 0660
```

---

## 8. Identityモデル

### 8.1 Principal ID

- Principal IDはUUIDv4である。
- 利用者がusernameを選択する機能は設けない。
- Principal IDはcanonical Git URLにも含まれるため、秘密値ではない。

### 8.2 Firebase project

- Firebase projectは固定された1 projectのみである。
- expected projectおよびissuerはサーバー設定で固定する。
- DBへissuerやtenant列を持たせない。

### 8.3 Firebase UIDとの対応

Firebase UIDとPrincipal IDはSQLiteで対応付ける。

```text
auth_identities
    firebase_uid  PRIMARY KEY
    principal_id  UNIQUE NOT NULL REFERENCES principals(principal_id)
```

### 8.4 Principalの自動作成

明示的な登録APIは設けない。

認証済みREST APIへの初回アクセス時に、次を単一SQLite transactionで行う。

1. Firebase ID tokenを検証する。
2. `firebase_uid`の対応を検索する。
3. 未登録ならPrincipal UUIDv4を生成する。
4. `principals`へ挿入する。
5. `auth_identities`へ対応を挿入する。
6. transactionをcommitする。

同時初回アクセスによる重複作成は、DB制約とtransactionで防ぐ。

Firebase利用者の無効化・削除とPrincipal状態の関係は未決定である。

---

## 9. Personal Access Token

### 9.1 形式

```text
cgit_<tokenId>_<secret>
```

- `tokenId`: UUIDv4
- `secret`: 暗号学的乱数32 bytes
- secret encoding: paddingなしBase64url

例の構造:

```text
cgit_00000000-0000-4000-8000-000000000000_<base64url-secret>
```

### 9.2 保存方式

- PAT全体のSHA-256 hashを保存する。
- HMACではない。
- hashはSQLite `BLOB` 32 bytesとして保存する。
- tokenIdで候補行を検索する。
- 入力された完全PATのSHA-256を計算し、保存hashとconstant-time比較する。
- 平文PATは発行時に一度だけ表示する。
- 平文PATをDB、file、logへ保存してはならない。

32-byte random secretを含むため、通常のSHA-256 hash保存で十分な探索耐性を持つという設計判断である。

### 9.3 有効期限

- `expires_at_ms IS NULL`: 無期限
- integer値: その時刻以降は無効
- 既定値: 無期限
- 利用者は発行時に有効期限を設定できる設計とする。

PAT管理REST APIの具体的範囲は未決定である。

### 9.4 失効

- `revoked_at_ms`による論理失効とする。
- 失効行は保持する。
- 再有効化は行わない。
- 必要なら新しいPATを発行する。
- Principalごとの有効PAT数に上限を設けない。

### 9.5 HTTP Basic互換性

- username fieldは任意の空でない文字列を受け入れる。
- usernameは認証・認可に使用しない。
- password fieldに完全なPATを指定する。
- Principal解決はPATだけに基づく。

---

## 10. Repository識別と名前規則

### 10.1 Repository ID

- UUIDv4
- Principal IDとは独立して生成する。

### 10.2 Repository name

長さ:

- 1文字以上
- 100文字以下

許可文字:

```text
ASCII lowercase letters: a-z
digits: 0-9
hyphen: -
underscore: _
dot: .
```

正規化と制約:

- ASCII uppercase inputはlowercaseへ変換する。
- 先頭と末尾は英数字でなければならない。
- `.`および`..`は禁止する。
- 連続する`.`は禁止する。
- 末尾`.git`は禁止する。
- whitespaceは禁止する。
- `/`、`\`、control characterは禁止する。
- Unicodeは禁止する。
- 同一Principal内で一意とする。
- renameは初期実装で禁止する。

### 10.3 名前予約

Repository名はpurgeされるまで、全状態で予約される。

```text
creating
ready
deleting
deleted
failed
```

同一Principalが同名Repositoryを作成しようとした場合:

- HTTP `409 Conflict`
- error code: `REPOSITORY_NAME_ALREADY_EXISTS`
- owner本人への応答であるため、既存`repositoryId`と`status`をdetailsへ含めてよい。

名前を解放するのは管理者CLIによるpurgeだけである。

---

## 11. Repositoryの物理配置

### 11.1 ready

```text
/srv/custom-git-host/repositories/{principalId}/{repositoryName}.git
```

### 11.2 creating staging

```text
/srv/custom-git-host/staging/{repositoryId}.git
```

### 11.3 deleted quarantine

```text
/srv/custom-git-host/quarantine/{repositoryId}.git
```

quarantineではRepository nameではなくRepository IDを使用し、owner/name変更や重複に依存しない保持を行う。

---

## 12. Repositoryライフサイクル

### 12.1 状態

```text
creating
ready
deleting
deleted
failed
```

### 12.2 failure情報

DB上では少なくとも次を保持する。

```text
failure_operation: create | delete | NULL
failure_code:      sanitized machine-readable code | NULL
failure_message:   sanitized user-facing message | NULL
```

Repository API表現では、`status = failed`の場合だけ次のobjectを必須で返す。

```json
{
  "failure": {
    "operation": "create",
    "code": "REPOSITORY_INITIALIZATION_FAILED",
    "message": "The repository could not be initialized."
  }
}
```

その他のstatusでは`failure` field自体を省略する。

次を外部へ公開してはならない。

- 絶対filesystem path
- SQLiteの生error本文
- 実行command line
- stack trace
- secret
- Authorization header

### 12.3 Repository作成

Repository作成は同期処理である。

概念手順:

1. REST requestを検証する。
2. DB transactionで`creating`行を作成する。
3. staging pathが存在しないことを確認する。
4. stagingでempty bare repositoryを初期化する。
5. symbolic `HEAD`を`refs/heads/main`へ設定する。
6. README、LICENSE、`.gitignore`、initial commitは作成しない。
7. owner Principal directoryを必要に応じて作成する。
8. stagingからready pathへatomic renameする。
9. DBを`ready`へ更新する。
10. `201 Created`を返す。

Repository作成時はvisibilityを必須入力とし、既定値を設けない。

途中で失敗した場合:

- Repository行を削除しない。
- `failed`へ遷移する。
- `failure_operation = create`を記録する。
- 残存物を自動上書きしない。
- HTTP requestには適切な4xxまたは5xxを返す。

プロセスクラッシュ時の自動復旧方針は未決定である。

### 12.4 Repository削除

通常削除は論理削除であり、同期処理である。

概念手順:

1. ownerと`ready`状態を確認する。
2. DBを`deleting`へ更新する。
3. ready pathからquarantine pathへatomic renameする。
4. DBを`deleted`へ更新する。
5. `updatedAtMs`を削除完了時刻にする。
6. `200 OK`と削除後の完全なRepository表現を返す。

bare repositoryは無期限保持する。自動purgeは行わない。

### 12.5 DELETEの状態別挙動

| 現在状態 | 挙動 |
|---|---|
| `ready` | 同期論理削除を実行し、`200`と`status: deleted`を返す |
| `deleted` | 冪等に`200`と現在表現を返す。file操作・DB更新・`updatedAtMs`変更なし |
| `creating` | `409 Conflict`。変更なし |
| `deleting` | `409 Conflict`。変更なし |
| `failed` | `409 Conflict`。変更なし |

異常状態の修復、再削除、強制削除は管理者CLIの責務とする。

### 12.6 Restoreとpurge

- restoreは管理者CLIだけが行える。
- purgeは管理者CLIだけが行える。
- owner向けrestore REST APIは設けない。
- admin REST APIは初期実装で設けない。
- purge後にのみRepository名を再利用できる。

Restore時の詳細な衝突処理は管理者CLI設計で確定する。

---

## 13. Default branchとref方針

- Repository作成時のdefault branch名は`main`固定である。
- 初期commitは存在しない。
- `HEAD`は`refs/heads/main`を指す。
- ownerによるforce pushおよびnon-fast-forward更新を許可する。
- branch deletionを許可する。
- tag deletionを許可する。
- `main`の削除も許可する。
- `main`が存在しなくなってもDBの`defaultBranch`は`main`のままとする。
- Web UIは「default branch unavailable」と表示する。
- Contents APIはdefault branchが存在しない場合にerrorを返す。
- Branch protectionは初期実装の対象外である。

---

## 14. Canonical Git HTTP URL

```text
https://{host}/git/{principalId}/{repositoryName}.git
```

- `.git` suffixは必須である。
- Principal IDはURLに含める。
- Repository nameは正規化済みの値を使用する。
- alias URLやusername-based URLは初期実装で設けない。

---

## 15. Git Smart HTTP

### 15.1 対応範囲

対応:

- `git-upload-pack`
- `git-receive-pack`
- Git protocol v2
- v2非対応クライアントの旧protocol fallback

非対応:

- Dumb HTTP
- Git archive over HTTP
- Repository内部fileの直接配信
- 未知のGit service
- 未知のGit path

### 15.2 許可する4パターン

READ:

```text
GET  /git/{principalId}/{repositoryName}.git/info/refs
     ?service=git-upload-pack

POST /git/{principalId}/{repositoryName}.git/git-upload-pack
```

WRITE:

```text
GET  /git/{principalId}/{repositoryName}.git/info/refs
     ?service=git-receive-pack

POST /git/{principalId}/{repositoryName}.git/git-receive-pack
```

以下は`404 Not Found`で拒否する。

- 未知のservice
- serviceなしの`info/refs`
- serviceの重複指定
- 余分なquery parameter
- 余分なpath segment
- methodとpathの不正な組み合わせ
- 未対応HTTP method
- Dumb HTTP path

### 15.3 Query検証

`info/refs`では`service` query parameterがちょうど1個であり、値が次のいずれかでなければならない。

```text
git-upload-pack
git-receive-pack
```

POST endpointではquery stringを認めない。

### 15.4 Git protocol v2

- クライアントの`Git-Protocol` headerをサポートする。
- headerなしの場合は旧protocolへfallbackする。
- Node.jsはheaderの長さ、control character、NUL、改行、基本構文を検証する。
- `key[=value]`のcolon区切りとして構文検証する。
- 未知のkey/valueは将来互換性のため許可する。
- 有効な値は内容と順序を変えずに`git-http-backend`へ渡す。
- 不正なheaderは`400 Bad Request`とする。

---

## 16. Git認証・認可規則

### 16.1 Public Repository READ

| Authorization状態 | 結果 |
|---|---|
| headerなし | 匿名READとして許可 |
| ownerの有効PAT | 許可 |
| 他Principalの有効PAT | 許可 |
| 形式不正PAT | PATを無視して匿名READとして許可 |
| hash不一致PAT | PATを無視して匿名READとして許可 |
| 失効PAT | PATを無視して匿名READとして許可 |
| 期限切れPAT | PATを無視して匿名READとして許可 |

これはcredential helperに古い資格情報が残っていてもpublic clone/fetchを失敗させないための互換性方針である。

無効PATの秘密値をlogへ記録してはならない。必要なら非secretの低優先度イベントだけを記録する。

### 16.2 Private Repository READ/WRITE

| 状態 | HTTP結果 |
|---|---|
| Authorizationなし | `401 Unauthorized` + `WWW-Authenticate: Basic realm="Git Access"` |
| 形式不正PAT | `401 Unauthorized` |
| hash不一致PAT | `401 Unauthorized` |
| 失効PAT | `401 Unauthorized` |
| 期限切れPAT | `401 Unauthorized` |
| 他Principalの有効PAT | `404 Not Found` |
| Repository不存在 | `404 Not Found` |
| ownerの有効PAT | 許可 |

他Principalの有効PATには`403`ではなく`404`を返し、private Repositoryの存在を秘匿する。

### 16.3 Public Repository WRITE

WRITEはvisibilityにかかわらずownerの有効PATを必要とする。他Principalの有効PATによるpublic Repository WRITEは許可しない。厳密なstatus codeは共通認可行列として実装時に整合させるが、private存在秘匿規則を破ってはならない。

### 16.4 Repository状態

Git Smart HTTPで配信できるのは`ready`だけである。

```text
creating / deleting / deleted / failed
    -> 配信しない
```

外部へ返すstatusは、存在秘匿と認証状態を考慮して決定する。

---

## 17. Nginx内部認可API

### 17.1 Path

Nginx internal location:

```text
/_internal/git-auth
```

Node.js endpoint:

```text
/api/v1/internal/git/auth
```

- Nginx locationには`internal`を指定する。
- 外部クライアントから直接呼び出せないこと。

### 17.2 NginxからNode.jsへ渡す情報

```text
X-Original-URI
X-Original-Method
Authorization
X-Original-Git-Protocol
```

- request bodyは内部認可APIへ渡さない。
- `Content-Length`は空にする。
- `X-Original-URI`には元のpathとqueryを含める。
- Node.jsがURIを一度だけ解析する。

### 17.3 成功応答

成功時:

```http
HTTP/1.1 204 No Content
```

常に返すheader:

```text
X-Git-Path-Info
X-Git-Query-String
```

認証済みの場合だけ返すheader:

```text
X-Authenticated-Principal
```

Git-Protocol headerが有効かつ存在する場合:

```text
X-Git-Protocol
```

返してはならない情報:

- 物理filesystem path
- Repository ID
- PAT token ID
- PAT hash
- Firebase UID
- secret

### 17.4 正規化済みPATH_INFO

例:

```text
/{principalId}/{repositoryName}.git/info/refs
/{principalId}/{repositoryName}.git/git-upload-pack
/{principalId}/{repositoryName}.git/git-receive-pack
```

Nginxは元URIからPATH_INFOを再構築せず、`X-Git-Path-Info`をそのままCGI `PATH_INFO`へ設定する。

### 17.5 正規化済みQUERY_STRING

```text
GET info/refs upload:
    service=git-upload-pack

GET info/refs receive:
    service=git-receive-pack

POST endpoints:
    empty string
```

Nginxは元query stringをFastCGIへ直接渡してはならない。

### 17.6 REMOTE_USER

認証済みの場合:

```text
REMOTE_USER=<Principal UUIDv4>
```

匿名public READ:

```text
REMOTE_USER未設定または空
```

人間向けusernameは使用しない。

---

## 18. git-http-backend CGI環境

最低限、Nginxは次を設定する。

```text
GIT_PROJECT_ROOT=/srv/custom-git-host/repositories
GIT_HTTP_EXPORT_ALL=1
PATH_INFO=<X-Git-Path-Info>
QUERY_STRING=<X-Git-Query-String>
REQUEST_METHOD=<original method>
CONTENT_TYPE=<original content type>
CONTENT_LENGTH=<original content length>
REMOTE_USER=<authenticated principal or empty>
HTTP_GIT_PROTOCOL=<validated Git-Protocol or empty>
```

必要に応じて標準CGI変数を追加するが、Git URLやRepository選択をNginx側で再解釈してはならない。

### 18.1 GIT_HTTP_EXPORT_ALL

```text
GIT_HTTP_EXPORT_ALL=1
```

- `git-daemon-export-ok` markerは使用しない。
- 公開可否と認可はNode.js内部認可APIへ一元化する。
- `/git/`配下のすべての許可対象requestへ`auth_request`を適用する。
- 認可API障害時はfail closedとする。

---

## 19. Request bufferingとresource制御

### 19.1 Body size

```nginx
client_max_body_size 0;
```

アプリケーション上のPOST bodyサイズ上限を設けない。

実際の上限は次に依存する。

- 一時filesystemの空き容量
- Repository filesystemの空き容量
- filesystem制限
- OS制限
- Nginx、fcgiwrap、Gitの実装制限

### 19.2 Request buffering

```nginx
fastcgi_request_buffering on;
```

NginxはGit POST bodyを完全に受信し、必要に応じて一時fileへ保存してからFastCGIへ渡す。

目的:

- 低速clientがfcgiwrap workerを長時間占有することを避ける。
- Nginx既定動作を利用し、初期構成を単純化する。

### 19.3 Body timeout

```nginx
client_body_timeout 10m;
```

これはpush全体の時間上限ではない。clientからの連続read間隔が10分間進まない場合に終了する。

### 19.4 FastCGI timeout

```nginx
fastcgi_send_timeout 30m;
fastcgi_read_timeout 30m;
```

- send timeout: NginxからFastCGIへの連続writeが30分進まない場合
- read timeout: FastCGIからNginxへの連続readが30分進まない場合
- 処理全体のwall-clock上限ではない。

### 19.5 一時file cleanup

```nginx
client_body_in_file_only off;
client_body_temp_path /var/lib/custom-git-host/nginx-client-body 1 2;
```

通常request終了時の削除はNginxへ任せる。

孤児file対策:

- host起動時に専用一時領域を全削除する。
- Nginx service開始前にも全削除する。
- cleanupは専用mount完了後、Nginx worker起動前に行う。
- Nginx稼働中の時刻ベース定期削除は行わない。
- 手動cleanupはNginx停止中だけ許可する。
- 使用容量、空き容量、inode使用率を監視する。

---

## 20. fcgiwrap / systemd socket activation

### 20.1 Socket

```text
/run/custom-git-host/git-http-backend.sock
```

概念的なsocket unit:

```ini
[Socket]
ListenStream=/run/custom-git-host/git-http-backend.sock
SocketUser=custom-git
SocketGroup=custom-git-fcgi
SocketMode=0660
DirectoryMode=0700
```

実際の`DirectoryMode`はruntime directory生成方式と整合させる。

### 20.2 Worker pool

- 初期値: 2 worker
- READ/WRITE共通pool
- 動的自動調整なし
- 管理者設定で変更可能

3件目以降はlisten socketの待ち行列で待機する。待ち行列上限と満杯時の扱いは実装詳細として別途設定する。

### 20.3 Service分離

- Node.js serviceとfcgiwrap serviceは別unitとする。
- 同一`custom-git` OSユーザーで実行する。
- systemdがsocketの作成、所有権、削除を管理する。

---

## 21. SQLiteとmigration

### 21.1 Driver

```text
better-sqlite3
```

ORMは使用しない。

### 21.2 Migration file

- 番号付きSQL fileを使用する。
- 専用migration CLIが順番に適用する。
- migration CLIはNode.js application起動前に実行する。
- systemd serviceの`ExecStartPre`から呼び出す。

例:

```text
migrations/
├── 0001_initial.sql
├── 0002_add_example.sql
└── ...
```

### 21.3 schema_migrations

最低限の情報:

```text
version
filename
checksum
applied_at_ms
```

- 適用済みmigration fileのchecksumが変化していた場合、起動を失敗させる。
- 適用済みmigrationを黙って再適用または上書きしてはならない。

### 21.4 Migration status

migration CLIは次のstatusのいずれかをatomicに公開する。

```text
ready
migration_required
migration_running
migration_failed
```

更新手順:

1. `/run/custom-git-host-status`内の一時fileへ完全なJSONを書く。
2. fileをflush/fsyncする。
3. 必要に応じてdirectoryをfsyncする。
4. `migration-status.json`へatomic renameする。

公開JSONへ次を含めてはならない。

- filesystem path
- stack trace
- SQL本文
- DB接続情報
- secret

Node.jsがmigration failureで起動できなくても、Nginxがstatusを返せる構成とする。

---

## 22. Repository管理REST API

### 22.1 認証とowner

- Firebase認証済みPrincipalだけが使用できる。
- ownerは認証済みPrincipalに暗黙固定する。
- `principalId`または`ownerPrincipalId`をURLやrequest bodyで指定させない。
- 管理者の代理操作は別の管理CLIまたは将来の管理APIとして設計する。

### 22.2 Endpoint

```text
POST   /api/v1/repositories
GET    /api/v1/repositories
GET    /api/v1/repositories/{repositoryId}
DELETE /api/v1/repositories/{repositoryId}
```

初期実装ではrename、owner restore、purgeをREST APIへ含めない。

---

## 23. Repository作成API

### 23.1 Request

```http
POST /api/v1/repositories
Content-Type: application/json
Authorization: Bearer <Firebase ID token>
```

```json
{
  "name": "example",
  "visibility": "private"
}
```

許可fieldは`name`と`visibility`だけである。

- `visibility`は必須
- 値は`public`または`private`
- 不明な追加fieldは`400 Bad Request`
- `repositoryId`はserverがUUIDv4で生成
- ownerは認証済みPrincipal
- default branchは`main`
- initial contentなし

### 23.2 Success

```http
HTTP/1.1 201 Created
Location: /api/v1/repositories/{repositoryId}
Content-Type: application/json
```

response bodyはRepository詳細APIと同じ完全表現を返す。

### 23.3 Duplicate name

```http
HTTP/1.1 409 Conflict
```

概念error:

```json
{
  "error": {
    "code": "REPOSITORY_NAME_ALREADY_EXISTS",
    "message": "A repository with this name already exists.",
    "details": {
      "repositoryId": "UUIDv4",
      "status": "deleted"
    }
  }
}
```

共通error envelopeの最終仕様は実装詳細として統一する。

---

## 24. Repository一覧API

### 24.1 Request

```text
GET /api/v1/repositories
GET /api/v1/repositories?includeDeleted=true
```

### 24.2 状態filter

既定で返す:

```text
creating
ready
deleting
failed
```

既定で除外:

```text
deleted
```

`includeDeleted=true`の場合は`deleted`も含める。

- `includeDeleted`は`true`または`false`だけを許可する。
- 不正値は`400 Bad Request`。

### 24.3 Pagination

初期実装ではpaginationを行わない。

以前検討した次の仕様は採用しない。

- `limit`
- `cursor`
- `nextCursor`
- HMAC署名付きcursor
- cursor有効期限

### 24.4 Sort

```text
createdAtMs DESC
repositoryId DESC
```

同一millisecondの作成でもRepository IDをtie-breakerとして安定した順序を得る。

### 24.5 Response

```json
{
  "repositories": [
    {
      "repositoryId": "UUIDv4",
      "ownerPrincipalId": "UUIDv4",
      "name": "example",
      "visibility": "private",
      "status": "ready",
      "defaultBranch": "main",
      "gitHttpUrl": "https://host/git/{principalId}/example.git",
      "createdAtMs": 1785140000000,
      "updatedAtMs": 1785140000000
    }
  ]
}
```

トップレベルをobjectとし、将来metadataやpaginationを後方互換的に追加できる形にする。初期実装では`repositories`だけを返す。

---

## 25. Repository詳細API

```text
GET /api/v1/repositories/{repositoryId}
```

ownerは全状態を取得できる。

```text
creating
ready
deleting
deleted
failed
```

`deleted`取得に`includeDeleted`は不要である。

他Principalが所有するRepository IDを指定した場合:

```text
404 Not Found
```

Repository不存在の場合も同じ`404`を返す。Repository IDの有効性や所有関係を秘匿する。

public Repositoryであっても、所有者向け管理APIから他Principalへ返さない。公開閲覧はGit Smart HTTPまたは将来の公開閲覧APIへ分離する。

---

## 26. Repository削除API

```text
DELETE /api/v1/repositories/{repositoryId}
```

成功時:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

削除後の完全なRepository表現を返す。

```json
{
  "repositoryId": "UUIDv4",
  "ownerPrincipalId": "UUIDv4",
  "name": "example",
  "visibility": "private",
  "status": "deleted",
  "defaultBranch": "main",
  "gitHttpUrl": "https://host/git/{principalId}/example.git",
  "createdAtMs": 1785140000000,
  "updatedAtMs": 1785141000000
}
```

`gitHttpUrl`は論理識別情報として残るが、`status: deleted`中は利用できない。

他Principal所有および不存在はともに`404 Not Found`とする。

状態別の詳細は「Repositoryライフサイクル」を参照する。

---

## 27. Repository resource表現

共通field:

```text
repositoryId
ownerPrincipalId
name
visibility
status
defaultBranch
gitHttpUrl
createdAtMs
updatedAtMs
```

`status = failed`の場合だけ:

```text
failure.operation
failure.code
failure.message
```

内部実装情報を外部resourceへ含めてはならない。

- physical path
- staging/quarantine path
- PAT token ID
- database rowid
- raw error

---

## 28. 暫定データモデル

この節は、確定した契約を実装可能な形に写像した暫定schemaである。列名の最終調整は可能だが、意味と制約を変える場合は設計変更として扱う。

### 28.1 principals

```sql
CREATE TABLE principals (
    principal_id   TEXT PRIMARY KEY,
    created_at_ms  INTEGER NOT NULL,
    updated_at_ms  INTEGER NOT NULL
    -- status関連は未決定
);
```

制約:

- `principal_id`はcanonical lowercase UUIDv4文字列
- timestampはUNIX epoch milliseconds

### 28.2 auth_identities

```sql
CREATE TABLE auth_identities (
    firebase_uid   TEXT PRIMARY KEY,
    principal_id   TEXT NOT NULL UNIQUE,
    created_at_ms  INTEGER NOT NULL,
    FOREIGN KEY (principal_id) REFERENCES principals(principal_id)
);
```

単一Firebase projectなのでissuer/tenant列は持たない。

### 28.3 repositories

```sql
CREATE TABLE repositories (
    repository_id       TEXT PRIMARY KEY,
    owner_principal_id  TEXT NOT NULL,
    name                TEXT NOT NULL,
    visibility          TEXT NOT NULL,
    status              TEXT NOT NULL,
    default_branch      TEXT NOT NULL,
    failure_operation   TEXT NULL,
    failure_code        TEXT NULL,
    failure_message     TEXT NULL,
    created_at_ms       INTEGER NOT NULL,
    updated_at_ms       INTEGER NOT NULL,
    FOREIGN KEY (owner_principal_id) REFERENCES principals(principal_id),
    UNIQUE (owner_principal_id, name)
);
```

期待するCHECK:

```text
visibility IN ('public', 'private')
status IN ('creating', 'ready', 'deleting', 'deleted', 'failed')
default_branch = 'main' in initial implementation
failure_operation IS NULL OR failure_operation IN ('create', 'delete')
```

failure列の整合性:

- `status = failed`ならoperation/code/messageを必須にする。
- その他statusではfailure列をNULLにする。
- DB CHECKまたはapplication invariantとして強制する。

`UNIQUE(owner_principal_id, name)`により、deletedやfailedを含む全状態で名前を予約する。

### 28.4 personal_access_tokens

```sql
CREATE TABLE personal_access_tokens (
    token_id            TEXT PRIMARY KEY,
    owner_principal_id  TEXT NOT NULL,
    token_hash          BLOB NOT NULL,
    expires_at_ms       INTEGER NULL,
    revoked_at_ms       INTEGER NULL,
    created_at_ms       INTEGER NOT NULL,
    FOREIGN KEY (owner_principal_id) REFERENCES principals(principal_id)
);
```

制約:

- `token_hash`は32 bytes
- token IDはUUIDv4
- 平文secret列は存在してはならない
- PAT表示名やlast-used timestampは未決定

### 28.5 schema_migrations

```sql
CREATE TABLE schema_migrations (
    version        INTEGER PRIMARY KEY,
    filename       TEXT NOT NULL UNIQUE,
    checksum       TEXT NOT NULL,
    applied_at_ms  INTEGER NOT NULL
);
```

checksum encodingとalgorithmはmigration CLIで固定し、変更しない。

### 28.6 未確定table

次は初期実装へ含めるか未決定である。

- audit log
- operation journal
- background jobs
- API idempotency keys

---

## 29. Transactionとatomicity

### 29.1 原則

SQLite transactionとfilesystem renameを単一のACID transactionにすることはできない。そのため、状態machineをtransaction boundaryとして使用する。

### 29.2 作成

- DBに`creating`をcommitしてからfilesystem作業を行う。
- ready rename後にDBを`ready`へ更新する。
- 中間状態は検査可能に残す。
- 既存pathを上書きしない。

### 29.3 削除

- DBを`deleting`へ更新する。
- quarantineへatomic renameする。
- DBを`deleted`へ更新する。
- 中間状態の自動復旧方針は今後決定する。

### 29.4 Path safety

- filesystem pathは検証済みPrincipal IDと正規化済みRepository nameからだけ生成する。
- client提供文字列を直接pathへ連結しない。
- path traversal、encoded slash、backslash、NUL、Unicode normalization差を拒否する。
- symlinkをRepository root解決へ使用しない。

---

## 30. 管理者CLI

初期実装では少なくとも次の責務を持つ。

### 30.1 Migration

- pending migrationの検出
- checksum検証
- migration適用
- migration status更新

### 30.2 Restore

- `deleted` Repositoryをquarantineからready領域へ戻す。
- DBを`ready`へ戻す。
- owner/name衝突を検査する。
- 一般利用者へ直接公開しない。

### 30.3 Purge

- Repository metadataとquarantine/staging残存物を明示的に物理削除する。
- 名前を解放する。
- 対象Repository IDを明示的に要求する。
- 誤操作防止を備える。

### 30.4 Abnormal-state repair

`creating`、`deleting`、`failed`の検査と修復を担う。具体的な自動判定・command設計は未決定である。

---

## 31. systemd構成の概念

unit名は暫定である。

```text
custom-git.service
custom-git-fcgi.socket
custom-git-fcgi.service
custom-git-clean-client-body.service
```

### custom-git.service

- User=`custom-git`
- Node.js application
- loopback only
- `ExecStartPre`でmigration CLIを実行
- migration failure時はNode.jsを起動しない

### custom-git-fcgi.socket

- FastCGI Unix socket管理
- `custom-git:custom-git-fcgi`, `0660`

### custom-git-fcgi.service

- User=`custom-git`
- socket activation
- fcgiwrap worker数2

### cleanup service

- Nginx停止中またはNginx起動前にclient body専用領域を空にする。
- 専用mount後に実行する。
- 稼働中の定期削除は行わない。

systemd hardening directiveの詳細は実装時に安全な既定値としてまとめて設定する。

---

## 32. Security要件

### 32.1 Secret logging禁止

次をlogへ記録してはならない。

- Authorization header
- PAT plaintext
- Firebase ID token
- Firebase credential secret
- SQLite内のtoken hash

PAT token IDも通常logへ不要であり、監査要件が決まるまでは外部応答へ返さない。

### 32.2 Internal API保護

- Nginx internal locationだけから呼び出す。
- Node.jsはloopbackだけでlistenする。
- 内部APIはrequest bodyを受け取らない。
- 必須header欠落は拒否する。
- 内部API応答にphysical pathを含めない。

### 32.3 Repository存在秘匿

- private Repositoryに他Principalの有効PATでアクセスした場合は`404`。
- owner向けRepository管理APIに他PrincipalのRepository IDを指定した場合も`404`。

### 32.4 Public READ互換性

public READでは無効credentialを匿名扱いへfallbackする。この例外はpublic READだけに限定し、WRITEやprivate accessへ拡張してはならない。

### 32.5 Local privilege

単一OSユーザー構成は初期実装の単純化である。Git backendが侵害された場合、同一ユーザーが読めるSQLiteや設定へ到達可能である。将来のhardeningとしてOSユーザー分離を検討する。

---

## 33. Migration status公開

未認証で公開してよい情報は、サービスがmigration上利用可能かどうかに限定する。

概念response:

```json
{
  "status": "ready",
  "updatedAtMs": 1785140000000
}
```

failure時もsanitizedなcode程度に留める。

```json
{
  "status": "migration_failed",
  "updatedAtMs": 1785140000000,
  "code": "MIGRATION_FAILED"
}
```

次を公開しない。

- migration filenameの絶対path
- SQL error本文
- stack trace
- database path
- command line

---

## 34. Web UI

現時点で確定している最小要件:

- migration状態を未認証で表示できる。
- default branchが存在しない場合、「default branch unavailable」を表示できる。

Repository一覧、PAT管理、Contents操作などのWeb UI範囲はAPI設計確定後に決定する。

---

## 35. 初期受入条件

以下は確定済み範囲に対する最低限の受入条件である。

### 35.1 Principal

- 有効なFirebase tokenで初回REST accessを行うとPrincipalが一度だけ作成される。
- 同時初回requestでもPrincipalが重複しない。

### 35.2 Repository作成

- `name`と`visibility`で空bare repositoryを作成できる。
- response時点でGit URLからclone discoveryが可能である。
- `HEAD`が`refs/heads/main`を指す。
- initial commitが存在しない。
- 同名作成が全状態で`409`になる。

### 35.3 Git READ

- public Repositoryを匿名clone/fetchできる。
- public Repositoryへ無効PATを送っても匿名READへfallbackする。
- private Repositoryはowner PATでclone/fetchできる。
- private Repositoryへ認証なしでアクセスすると`401`とBasic challengeを返す。
- private Repositoryへ他Principal PATでアクセスすると`404`を返す。

### 35.4 Git WRITE

- owner PATでpushできる。
- force pushできる。
- branch、tag、`main`を削除できる。
- 他Principalはpublic Repositoryにもpushできない。

### 35.5 Protocol

- protocol v2 clientが動作する。
- headerなしの旧protocol clientも動作する。
- 不正なGit-Protocol headerは`400`になる。
- Dumb HTTP pathは`404`になる。

### 35.6 Delete

- ready Repositoryを削除するとquarantineへ移動し`deleted`になる。
- 削除済みRepositoryへ再DELETEしても追加変更せず`200`になる。
- deleted Repository名を再利用できない。
- purgeするまでbare repositoryが保持される。

### 35.7 Migration

- 未適用migrationを順番に適用できる。
- 適用済みmigration fileを変更すると起動失敗する。
- Node.jsが起動しなくてもmigration statusを確認できる。

### 35.8 Isolation

- NginxユーザーはFastCGI socketおよび公開status file以外のprivate dataを直接読めない。
- staging/quarantineはGit HTTP経由で配信されない。

---

## 36. 撤回・置換された旧方針

実装者が過去案を誤って採用しないよう、明示的に記録する。

### 36.1 OSユーザー分離

撤回:

```text
custom-git-app
custom-git-git
custom-git-repo共有group
setgid
core.sharedRepository=group
```

現行:

```text
単一OSユーザー custom-git
Node.jsとfcgiwrapは別systemd service
```

### 36.2 Repository一覧pagination

撤回:

```text
cursor pagination
limit
nextCursor
HMAC-signed cursor
cursor expiry
```

現行:

```text
初期実装ではpaginationなし
createdAtMs DESC, repositoryId DESC
```

### 36.3 Repository restore

撤回候補ではなく、明示的な現行方針:

```text
owner REST restoreなし
admin CLIのみ
```

### 36.4 PAT hash

現行:

```text
完全PATの通常SHA-256
HMACではない
```

### 36.5 Public READの無効PAT

現行:

```text
401ではなく匿名READへfallback
```

---

## 37. 残るブロッキング項目

以下は、初期実装の公開契約またはデータ整合性へ影響するため、実装前または該当機能着手前に決定する。

### 37.1 Contents APIの範囲

決定が必要な事項:

- 初期実装へ含めるoperation: GET / PUT / DELETE
- GitHub互換subsetとするか独自APIとするか
- default branch限定か、任意branch指定を許可するか
- directory listingとfile取得のresponse形式
- Base64 contentの扱い
- path、file size、symlinkの制限

### 37.2 Contents APIのwrite実装と同時更新

決定が必要な事項:

- Git plumbingを用いたcommit生成方式
- commit author / committer
- expected blob SHAまたはexpected commitによるoptimistic concurrency
- Git Smart HTTP pushとの同時更新排他
- Repository単位queueまたはcompare-and-swap

### 37.3 PAT管理REST API

決定が必要な事項:

- 発行、一覧、失効endpoint
- PAT label/nameの有無
- expiration入力形式
- last-used情報
- 発行時response
- Firebase再認証を要求するか

### 37.4 Principal無効化

決定が必要な事項:

- Firebase account disabled/deleted時の既存PAT
- Principal status列
- disabled PrincipalのREST/Git status code
- 管理者による停止・再開

### 37.5 Repository crash recovery

決定が必要な事項:

- 起動時に`creating`、`deleting`を自動検査するか
- DB状態とphysical pathの組み合わせごとの復旧規則
- 自動repairとadmin CLI repairの境界
- `failed`への自動遷移条件

### 37.6 Git hook方針

決定が必要な事項:

- hookを完全禁止するか
- 管理者管理hookだけを許可するか
- Repository内hook pathの扱い
- 将来のpolicy enforcementとの関係

ユーザー提供hookは安全上許可しない方向が妥当だが、まだ正式決定ではない。

### 37.7 監査log

決定が必要な事項:

- 初期実装へ含めるか
- Repository作成、削除、PAT発行・失効、Git WRITEの記録
- actor Principal、Repository ID、時刻、結果
- retention
- secretを記録しないschema

### 37.8 Backup consistency

決定が必要な事項:

- SQLiteとbare repositoryを整合した世代として取得する方式
- service停止backupかonline snapshotか
- quarantineをbackup対象に含めるか
- restore試験
- backup retention

---

## 38. 非ブロッキング実装課題

次は重要だが、公開契約を変えず推奨既定値として実装できるため、逐次設計質問の対象としない。

- 共通REST error envelopeの細部
- error code一覧
- Nginx FastCGI buffer response設定
- socket backlogの初期値
- systemd hardening directive
- structured logging format
- log rotation
- health check endpoint
- disk/inode monitoring method
- package layout
- TypeScript lint/test toolchain
- CI workflow
- exact CLI command names
- config file format

これらは本書の原則と矛盾しない形で実装し、必要に応じて追補する。

---

## 39. 実装開始時の推奨ディレクトリ構造

これは規範ではなく、責務分離を保つための推奨案である。

```text
.
├── README.md
├── DESIGN.md
├── package.json
├── tsconfig.json
├── migrations/
├── src/
│   ├── app/
│   ├── api/
│   │   ├── public/
│   │   └── internal/
│   ├── auth/
│   │   ├── firebase/
│   │   └── pat/
│   ├── repositories/
│   │   ├── domain/
│   │   ├── persistence/
│   │   └── filesystem/
│   ├── git-http/
│   ├── db/
│   ├── cli/
│   ├── config/
│   └── observability/
├── systemd/
├── nginx/
└── test/
    ├── unit/
    ├── integration/
    └── e2e/
```

重要なのは、Git URL解析と認可を`git-http`領域へ集約し、Nginx設定やfilesystem helperへ重複実装しないことである。

---

## 40. まとめ

本システムの初期アーキテクチャは次で固定されている。

```text
Nginx
  -> REST/Web: loopback Node.js
  -> Git auth: Node.js internal auth API
  -> Git data: systemd socket activated fcgiwrap -> git-http-backend

Identity:
  Firebase UID -> Principal UUIDv4

Git credential:
  PAT over HTTP Basic

Metadata:
  SQLite via better-sqlite3

Repository data:
  local bare repositories

Deletion:
  quarantine-based logical deletion

Deployment:
  single Linux host, direct installation, no Docker
```

残るブロッキング項目はContents API、PAT管理、Principal無効化、クラッシュ復旧、hook、監査、backupに限定する。それらを決定した後、本書を更新して初期実装仕様を凍結する。
