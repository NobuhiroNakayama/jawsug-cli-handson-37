このドキュメントは、JAWS-UG CLI専門支部 37 - CodeCommit入門のハンズオンの手順です。
https://jawsug-cli.doorkeeper.jp/events/33301

# 0. 前提条件

## 0.1. 権限

ハンズオンは以下の権限を付与されたユーザ（もしくはロール）で実行してください。

- CodeCommitに関するフルコントロール権限
- IAMに関するフルコントロール権限

## 0.2.テスト環境について

動作確認はAmazon Linuxで行っています。


## 0.3.動作環境確認

AWS CLIのバージョンを確認します。

```
aws --version
```

```
aws-cli/1.9.18 Python/2.7.10 Linux/4.1.13-19.30.amzn1.x86_64 botocore/1.3.18
```

プロファイルが想定のものになっていることを確認します。

```
aws configure list
```

```
Name                    Value             Type    Location
----                    -----             ----    --------
profile                <not set>             None    None
access_key     ****************R63A         iam-role
secret_key     ****************0xHB         iam-role
region                us-east-1              env    AWS_DEFAULT_REGION
```


# 1. 事前準備

## 1.1. リージョンの指定

リージョンを設定します。
現時点（2016年1月14日）では、バージニアのみサポート

```
export AWS_DEFAULT_REGION='us-east-1'
```

# 2. IAMユーザの作成

## 2.1. IAMユーザの作成

CodeCommitではSSHおよびHTTPS接続をサポートしています。
リポジトリへの接続時の認証のためにIAMユーザを作成します。

ユーザ名を設定します。

```
GITUSER='git-user'
```

同じ名前のユーザがいないことを確認します

```
aws iam get-user --user-name ${GITUSER}
```

```
A client error (NoSuchEntity) occurred when calling the GetUser operation: The user with name git-user cannot be found.
```

IAMユーザを作成します

```
aws iam create-user --user-name ${GITUSER}
```

```
{
    "User": {
        "UserName": "git-user",
        "Path": "/",
        "CreateDate": "2015-12-21T10:06:05.212Z",
        "UserId": "A********************",
        "Arn": "arn:aws:iam::************:user/git-user"
    }
}
```

# 3. 認証情報の設定（SSH接続用）

## 3.1. キーペアの作成

sshキーの名前を設定（既存ファイルと名前重複を避けるか、既存ファイルのリネームなど対策を！）

```
SSHKEYNAME='id_rsa'
```

sshキーの名前を確認

```
cat << ETX

   secretkey-name: ~/.ssh/${SSHKEYNAME}
   publickey-name: ~/.ssh/${SSHKEYNAME}.pub

ETX
```

```

   secretkey-name: ~/.ssh/id_rsa
   publickey-name: ~/.ssh/id_rsa.pub

```

sshキーを作成
（パスワードの設定は不要）

```
cd ~/.ssh
ssh-keygen -t rsa -b 2048 -f ${SSHKEYNAME}
```

```
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/ec2-user/.ssh/id_rsa.
Your public key has been saved in /home/ec2-user/.ssh/id_rsa.pub.
The key fingerprint is:
**:**:**:**:**:**:**:**:**:**:**:**:**:**:**:** ec2-user@ip-172-***-***-***
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```

出力されたファイルを確認

```
ls -al
```

```
total 24
drwx------ 2 ec2-user ec2-user 4096 Dec 23 14:20 .
drwx------ 8 ec2-user ec2-user 4096 Dec 21 10:30 ..
-rw------- 1 ec2-user ec2-user  391 Nov 17 07:12 authorized_keys
-rw------- 1 ec2-user ec2-user 1679 Dec 23 14:20 id_rsa
-rw-r--r-- 1 ec2-user ec2-user  407 Dec 23 14:20 id_rsa.pub
```

## 3.2. 公開鍵をアップロード

公開鍵をアップロードします。

```
aws iam upload-ssh-public-key --user-name ${GITUSER} --ssh-public-key-body file://${SSHKEYNAME}.pub
```

```
{
    "SSHPublicKey": {
        "UserName": "git-user",
        "Status": "Active",
        "SSHPublicKeyBody": "ssh-rsa **************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************** ec2-user@ip-172-***-***-***\n",
        "UploadDate": "2015-12-21T10:10:54.908Z",
        "Fingerprint": "**:**:**:**:**:**:**:**:**:**:**:**:**:**:**:**",
        "SSHPublicKeyId": "********************"
    }
}
```

確認

```
aws iam list-ssh-public-keys --user-name ${GITUSER}
```

```
{
    "SSHPublicKeys": [
        {
            "UserName": "git-user",
            "Status": "Active",
            "SSHPublicKeyId": "********************",
            "UploadDate": "2016-01-03T07:56:21Z"
        }
    ],
    "IsTruncated": false
}
```

## 3.3. 認証用設定ファイルの作成

公開鍵のIDを取得
（有効なSSHキーが一つだけ設定されている前提）

```
SSHPUBID=`aws iam list-ssh-public-keys --user-name ${GITUSER} --query 'SSHPublicKeys[0].SSHPublicKeyId' --output text`

cat << ETX

   SSHPublicKeyId: ${SSHPUBID}

ETX
```

```

   SSHPublicKeyId: ********************

```

設定ファイルのリネーム（既存の設定ファイルがある場合）

```
ls ~/.ssh/config.old
sudo cp ~/.ssh/config ~/.ssh/config.old
```

設定ファイルを作成

```
cat << EOF > ~/.ssh/config
Host git-codecommit.*.amazonaws.com
  User ${SSHPUBID}
  IdentityFile ~/.ssh/${SSHKEYNAME}
EOF

cat ~/.ssh/config
```

```
Host git-codecommit.*.amazonaws.com
  User ********************
  IdentityFile ~/.ssh/id_rsa
```

認証情報の設定ファイルを確認

```
ls -al config
```

```
-rw-rw-r-- 1 ec2-user ec2-user   93 Dec 23 14:49 config
```

設定ファイルのパーミッションを変更

```
chmod 600 config
```

設定ファイルのパーミッションを確認

```
ls -al config
```

```
-rw------- 1 ec2-user ec2-user 93 Dec 23 14:49 config
```

## 3.4. 接続確認

```
ssh git-codecommit.us-east-1.amazonaws.com
```

yesを入力

```
The authenticity of host 'git-codecommit.us-east-1.amazonaws.com (72.21.203.185)' can't be established.
RSA key fingerprint is **:**:**:**:**:**:**:**:**:**:**:**:**:**:**:**.
Are you sure you want to continue connecting (yes/no)?
```

```
Warning: Permanently added 'git-codecommit.us-east-1.amazonaws.com,72.21.203.185' (RSA) to the list of known hosts.
You have successfully authenticated over SSH. You can use Git to interact with AWS CodeCommit. Interactive shells are not supported.Connection to git-codecommit.us-east-1.amazonaws.com closed by remote host.
Connection to git-codecommit.us-east-1.amazonaws.com closed.
```

# 4. 権限を付与

マネージドポリシーのARNを決定

```
ARN="arn:aws:iam::aws:policy/AWSCodeCommitPowerUser"
```

ARNを確認

```
cat << ETX

   iam-user-name: ${GITUSER}
   ManagedPolicyARN: ${ARN}

ETX
```

```

   iam-user-name: git-user
   ManagedPolicyARN: arn:aws:iam::aws:policy/AWSCodeCommitPowerUser

```

マネージドポリシーをアタッチ

```
aws iam attach-user-policy --user-name ${GITUSER} --policy-arn ${ARN}
```

ポリシーがアタッチされたことを確認

```
aws iam list-attached-user-policies --user-name ${GITUSER}
```

```
{
    "AttachedPolicies": [
        {
            "PolicyName": "AWSCodeCommitPowerUser",
            "PolicyArn": "arn:aws:iam::aws:policy/AWSCodeCommitPowerUser"
        }
    ]
}
```

# 5. Gitをインストール

インストール（インストールされていない場合）

```
sudo yum install git -y
```

```
Loaded plugins: priorities, update-motd, upgrade-helper
amzn-main/latest                                         | 2.1 kB     00:00
amzn-updates/latest                                      | 2.3 kB     00:00
Resolving Dependencies
--> Running transaction check
---> Package git.x86_64 0:2.4.3-7.42.amzn1 will be installed
--> Processing Dependency: perl-Git = 2.4.3-7.42.amzn1 for package: git-2.4.3-7.42.amzn1.x86_64
--> Processing Dependency: perl(Term::ReadKey) for package: git-2.4.3-7.42.amzn1.x86_64
--> Processing Dependency: perl(Git) for package: git-2.4.3-7.42.amzn1.x86_64
--> Processing Dependency: perl(Error) for package: git-2.4.3-7.42.amzn1.x86_64
--> Running transaction check
---> Package perl-Error.noarch 1:0.17020-2.9.amzn1 will be installed
---> Package perl-Git.noarch 0:2.4.3-7.42.amzn1 will be installed
---> Package perl-TermReadKey.x86_64 0:2.30-20.9.amzn1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package              Arch       Version                 Repository        Size
================================================================================
Installing:
 git                  x86_64     2.4.3-7.42.amzn1        amzn-updates     9.8 M
Installing for dependencies:
 perl-Error           noarch     1:0.17020-2.9.amzn1     amzn-main         33 k
 perl-Git             noarch     2.4.3-7.42.amzn1        amzn-updates      61 k
 perl-TermReadKey     x86_64     2.30-20.9.amzn1         amzn-main         33 k

Transaction Summary
================================================================================
Install  1 Package (+3 Dependent packages)

Total download size: 10 M
Installed size: 24 M
Downloading packages:
(1/4): git-2.4.3-7.42.amzn1.x86_64.rpm                   | 9.8 MB     00:00
(2/4): perl-Error-0.17020-2.9.amzn1.noarch.rpm           |  33 kB     00:00
(3/4): perl-Git-2.4.3-7.42.amzn1.noarch.rpm              |  61 kB     00:00
(4/4): perl-TermReadKey-2.30-20.9.amzn1.x86_64.rpm       |  33 kB     00:00
--------------------------------------------------------------------------------
Total                                               25 MB/s |  10 MB  00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : 1:perl-Error-0.17020-2.9.amzn1.noarch                        1/4
  Installing : perl-TermReadKey-2.30-20.9.amzn1.x86_64                      2/4
  Installing : perl-Git-2.4.3-7.42.amzn1.noarch                             3/4
  Installing : git-2.4.3-7.42.amzn1.x86_64                                  4/4
  Verifying  : git-2.4.3-7.42.amzn1.x86_64                                  1/4
  Verifying  : 1:perl-Error-0.17020-2.9.amzn1.noarch                        2/4
  Verifying  : perl-Git-2.4.3-7.42.amzn1.noarch                             3/4
  Verifying  : perl-TermReadKey-2.30-20.9.amzn1.x86_64                      4/4

Installed:
  git.x86_64 0:2.4.3-7.42.amzn1

Dependency Installed:
  perl-Error.noarch 1:0.17020-2.9.amzn1      perl-Git.noarch 0:2.4.3-7.42.amzn1
  perl-TermReadKey.x86_64 0:2.30-20.9.amzn1

Complete!
```

Gitのバージョンを確認

```
git --version
```

```
git version 2.4.3
```

# 6. 環境設定 (Git)

## 6.1. ユーザ名およびメールアドレスの設定

ユーザ名とメールアドレスを変数に設定

```
MYNAME=user01
EMAIL=user01@example.com
```

確認

```
cat << ETX

   name: ${MYNAME}
   e-mail: ${EMAIL}

ETX
```

```

   name: user01
   e-mail: user01@example.com

```

ユーザ名とメールアドレスを設定

```
git config --global user.name "${MYNAME}"
git config --global user.email "${EMAIL}"
```

確認

```
git config --get user.name
git config --get user.email
```

```
user01
user01@example.com
```

# 7. ローカルリポジトリの作成

## 7.1. ディレクトリの作成

ディレクトリの作成

```
cd ~
mkdir ${REPONAME}
cd ${REPONAME}
```

## 7.2. 初期化

```
git init
```

```
Initialized empty Git repository in /home/ec2-user/MyDemoRepo/.git/
```

# 8. コンテンツの追加

## 8.1. インデックスの状態を確認

```
git status
```

```
On branch master

Initial commit

nothing to commit (create/copy files and use "git add" to track)
```

## 8.2. コンテンツを追加

ファイル名を決定

```
FILENAME=codecommit.txt
```

ファイルを作成

```
touch ${FILENAME}
```

ファイルを追加

```
git add ${FILENAME}
```

## 8.3. 変更をコミット

コミット

```
git commit -m "create ${FILENAME}"
```

```
[master (root-commit) 0fd2151] create codecommit.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 codecommit.txt
```

確認

```
git log
```

## 8.4. タグを付与

バージョン番号を設定

```
git tag "v0.1.0"
```

確認

```
git show v0.1.0
```

```
commit bb4bd3bbbf3152dc6cd16783ff073a6b38471605
Author: user01 <user01@example.com>
Date:   Sun Jan 10 06:50:26 2016 +0000

    create codecommit.txt

diff --git a/codecommit.txt b/codecommit.txt
new file mode 100644
index 0000000..e69de29
```


# 9. リモートリポジトリの作成

リポジトリ名および説明を設定します
（リポジトリ名はグローバルでユニークである必要はありません）

```
REPONAME='my-demo-repo'
REPODESC='My demonstration repository'
```

変数を確認します

```
cat << ETX

   repository-name: ${REPONAME}
   repository-description: ${REPODESC}

ETX
```

```

   repository-name: MyDemoRepo
   repository-description: My demonstration repository

```

リポジトリを作成します。

```
aws codecommit create-repository --repository-name ${REPONAME} --repository-description "${REPODESC}"
```

```
{
    "repositoryMetadata": {
        "repositoryName": "MyDemoRepo",
        "cloneUrlSsh": "ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo",
        "lastModifiedDate": 1450690437.262,
        "repositoryDescription": "My demonstration repository",
        "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo",
        "creationDate": 1450690437.262,
        "repositoryId": "********-****-****-****-************",
        "Arn": "arn:aws:codecommit:us-east-1:************:MyDemoRepo",
        "accountId": "************"
    }
}
```

リポジトリの一覧を表示します。

```
aws codecommit list-repositories
```

```
{
    "repositories": [
        {
            "repositoryName": "MyDemoRepo",
            "repositoryId": "********-****-****-****-************"
        }
    ]
}
```

作成したリポジトリを表示します

```
aws codecommit get-repository --repository-name ${REPONAME}
```

```
{
    "repositoryMetadata": {
        "repositoryName": "MyDemoRepo",
        "cloneUrlSsh": "ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo",
        "lastModifiedDate": 1450690437.262,
        "repositoryDescription": "My demonstration repository",
        "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo",
        "creationDate": 1450690437.262,
        "repositoryId": "********-****-****-****-************",
        "Arn": "arn:aws:codecommit:us-east-1:************:MyDemoRepo",
        "accountId": "************"
    }
}
```


# 10. 変更をプッシュ

プッシュ前の状態を確認（リポジトリは空の状態）

```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name  master
```

```
A client error (BranchDoesNotExistException) occurred when calling the GetBranch operation: None
```

URL(SSH)を確認

```
SSHURL=`aws codecommit get-repository --repository-name ${REPONAME} --query repositoryMetadata.cloneUrlSsh --output text`

echo ${SSHURL}
```

```
ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
```

リモートリポジトリを設定

```
git remote add origin ${SSHURL}
```

確認

```
git remote -v
```

```
origin  ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo (fetch)
origin  ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo (push)
```

コミットしたコンテンツをプッシュ

```
git push origin master
```

```
Counting objects: 3, done.
Writing objects: 100% (3/3), 216 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
To ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
 * [new branch]      master -> master
```

プッシュ後の状態を確認

```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name  master
```

```
{
    "branch": {
        "commitId": "****************************************",
        "branchName": "master"
    }
}
```

# 11. ブランチの作成、コンテンツの変更、マスターブランチへのマージ

## 11.1. ブランチの作成

コミットIDの確認

```
COMMITID=`aws codecommit get-branch --repository-name ${REPONAME} --branch-name master --query branch.commitId --output text`
```

ブランチ名の決定

```
BRANCHNAME="develop"
```

確認

```
cat << ETX

   Repository-Name: ${REPONAME}
   Commit ID: ${COMMITID}
   Branch Name: ${BRANCHNAME}

ETX
```

```

   Repository-Name: MyDemoRepo
   Commit ID: ****************************************
   Branch Name: MyNewBranch

```

ブランチの作成

```
aws codecommit create-branch --repository-name ${REPONAME} --branch-name ${BRANCHNAME} --commit-id ${COMMITID}
```

確認

```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name ${BRANCHNAME}
```

```
{
    "branch": {
        "commitId": "****************************************",
        "branchName": "MyNewBranch"
    }
}
```

```
aws codecommit list-branches --repository-name ${REPONAME}
```

```
{
    "branches": [
        "develop",
        "master"
    ]
}
```

リモートリポジトリからローカルリポジトリへ新しいブランチのコンテンツをプル

```
git pull origin ${BRANCHNAME}
```

```
From ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo
 * branch            develop    -> FETCH_HEAD
 * [new branch]      develop    -> origin/develop
Already up-to-date.
```


ブランチのスイッチ

```
git checkout ${BRANCHNAME}
```

```
Branch develop set up to track remote branch develop from origin.
Switched to a new branch 'develop'
```

確認1

```
git branch
```

```
* develop
  master
```

確認2

```
git status
```

```
On branch develop
Your branch is up-to-date with 'origin/develop'.
nothing to commit, working directory clean
```

## 11.2. コンテンツの修正

```
cat << EOF >> ${FILENAME}
fix for development
EOF

cat ${FILENAME}
```

```
fix for development
```

追加

```
git add ${FILENAME}
```

コミット

```
git commit -m "fix for development"
```

```
[MyNewBranch 3e121a8] add new line
 1 file changed, 1 insertion(+)
```

## 11.3. developブランチのプッシュ

```
git push origin ${BRANCHNAME}
```

```
Counting objects: 3, done.
Writing objects: 100% (3/3), 268 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
To ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo
   bb4bd3b..58880fe  develop -> develop
```

## 11.4. マスターブランチへのマージ

一般的なGit FlowではReleaseブランチを経由しますが、手順を簡略化するため、masterブランチへ直接マージします。

```
git checkout master
```

```
Switched to branch 'master'
```

確認

```
git branch
```

```
MyNewBranch
* master
```

マージ

```
git merge ${BRANCHNAME}
```

```
Updating 0fd2151..3e121a8
Fast-forward
 codecommit.txt | 1 +
 1 file changed, 1 insertion(+)
```

確認

```
git log
```

```
commit ****************************************
Author: user01 <user01@example.com>
Date:   Sun Jan 3 09:01:28 2016 +0000

    add new line

commit ****************************************
Author: user01 <user01@example.com>
Date:   Sun Jan 3 08:22:00 2016 +0000

    create codecommit.txt
```

## 11.5. リモートリポジトリへのプッシュ

```
git push origin master
```

```
Counting objects: 3, done.
Writing objects: 100% (3/3), 262 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
To ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
   0fd2151..3e121a8  master -> master
   0fd2151..3e121a8  MyNewBranch -> MyNewBranch
```

確認1

```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name  master
```

```
{
    "branch": {
        "commitId": "****************************************",
        "branchName": "master"
    }
}
```

確認2
```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name  ${BRANCHNAME}
```

```
{
    "branch": {
        "commitId": "****************************************",
        "branchName": "develop"
    }
}
```

## 11.6. タグを付与

バージョン番号を設定

```
git tag "v1.0.0"
```

確認

```
git show v1.0.0
```

```
commit 58880fea8cce5f5326f702b572665c0af31d4eb5
Author: user01 <user01@example.com>
Date:   Sun Jan 10 07:09:20 2016 +0000

    fix for development

diff --git a/codecommit.txt b/codecommit.txt
index e69de29..a4b7b23 100644
--- a/codecommit.txt
+++ b/codecommit.txt
@@ -0,0 +1 @@
+fix for development
```


# 12. 認証情報の設定（HTTPS接続用）

認証用のIAMユーザとして、SSH用に作成したIAMユーザを流用します。

## 12.1. アクセスキーおよびアクセスシークレットを作成

ユーザ名を確認します。

```
cat << ETX

        IAM_USER_NAME:  ${GITUSER}

ETX
```

access keyの作成と取得を行います。

```
cd ~

aws iam create-access-key \
        --user-name ${GITUSER} \
        > ${GITUSER}.json \
          && cat ${GITUSER}.json
```

```
{
    "AccessKey": {
        "UserName": "git-user",
        "Status": "Active",
        "CreateDate": "2016-01-14T15:12:22.319Z",
        "SecretAccessKey": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
        "AccessKeyId": "AKIAXXXXXXXXXXXXXXXX"
    }
}
```

sourceディレクトリ作成

```
mkdir -p ~/.aws/source
```

rcファイル作成

```
cat ${GITUSER}.json |\
        jq '.AccessKey | {AccessKeyId, SecretAccessKey}' |\
        sed '/[{}]/d' | sed 's/[\" ,]//g' | sed 's/:/=/' |\
        sed 's/AccessKeyId/aws_access_key_id/' |\
        sed 's/SecretAccessKey/aws_secret_access_key/' \
        > ~/.aws/source/${GITUSER}.rc \
        && cat ~/.aws/source/${GITUSER}.rc
```

```
aws_access_key_id=AKIAXXXXXXXXXXXXXXXX
aws_secret_access_key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

configファイル(単体)作成

```bash:変数の設定:
REGION_AWS_CONFIG="${AWS_DEFAULT_REGION}"

FILE_USER_CONFIG="${HOME}/.aws/source/${GITUSER}.config"

echo "[profile ${GITUSER}]" > ${FILE_USER_CONFIG} \
  && echo "region=${REGION_AWS_CONFIG}" >> ${FILE_USER_CONFIG} \
  && echo "" >> ${FILE_USER_CONFIG} \
  && cat ${FILE_USER_CONFIG}
```

```
[profile git-user]
region=us-east-1
```

~/.aws/credentials作成

既存の認証情報のバックアップを行います。

```
cp ~/.aws/credentials ~/.aws/credentials.old
```

認証情報を作成します。

```
file="${HOME}/.aws/credentials"
if [ -e ${file} ]; then mv ${file} ${file}.bak; fi
for i in `ls ${HOME}/.aws/source/*.rc`; do \
        name=`echo $i | sed 's/^.*\///' | sed 's/\.rc$//'` \
        && echo "[$name]" >> ${file} \
        && cat $i >> ${file} \
        && echo "" >> ${file} ;done \
        && cat ${file}
```

```
[git-user]
aws_access_key_id=AKIAXXXXXXXXXXXXXXXX
aws_secret_access_key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```


~/.aws/config作成

設定ファイルをバックアップします。

```
cp ~/.aws/config ~/.aws/config.old
```

設定ファイルを作成します。

```
cat ${HOME}/.aws/source/*.config > ${HOME}/.aws/config \
        && cat ${HOME}/.aws/config
```

```
[profile git-user]
region=us-east-1
```

## 12.2. ユーザの切り替え

現在のユーザを確認します。

```
aws configure list | grep profile
```

```
   profile                <not set>             None    None
```

切替後ユーザの確認

```
cat << ETX

        new:     IAM_USER_NAME:       ${GITUSER}

ETX
```


ユーザの切り替え

```
export AWS_DEFAULT_PROFILE=${GITUSER}
echo ${AWS_DEFAULT_PROFILE}
```

確認

```
aws configure list
```

```
Name                    Value             Type    Location
----                    -----             ----    --------
profile                 git-user           manual    --profile
access_key     ****************BX4Q shared-credentials-file
secret_key     ****************pUaQ shared-credentials-file
region                us-east-1              env    AWS_DEFAULT_REGION
```

コマンドテスト

```
aws codecommit list-repositories
```

```
{
    "repositories": [
        {
            "repositoryName": "my-demo-repo",
            "repositoryId": "53b8501f-f5df-47c7-bee0-817624ad8677"
        }
    ]
}
```

認証ファイルの削除

```
rm ${GITUSER}.json
```

## 12.3. 認証情報の設定

```
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

## 12.4. リモートリポジトリのエンドポイントの変更

URL(HTTPS)を確認

```
HTTPURL=`aws codecommit get-repository --repository-name ${REPONAME} --query repositoryMetadata.cloneUrlHttp --output text`

echo ${HTTPURL}
```

```
https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo
```

リモートリポジトリを設定

```
git remote add https ${HTTPURL}
```

確認

```
git remote -v
```

```
https   https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo (fetch)
https   https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo (push)
origin  ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo (fetch)
origin  ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo (push)
```

## 12.5. 動作確認

コンテンツの修正を行います。

```
cat << EOF >> ${FILENAME}
test HTTPS connection
EOF

cat ${FILENAME}
```

```
fix for development
test HTTPS connection
```

追加

```
git add ${FILENAME}
```

コミット

```
git commit -m "test HTTPS connection"
```

```
[master 53092be] test HTTPS connection
 1 file changed, 1 insertion(+)
```

Masterブランチへのプッシュ

```
git push https master
```

```
Counting objects: 3, done.
Writing objects: 100% (3/3), 294 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
To https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-demo-repo
   bf9fca4..53092be  master -> master
```


# 13. 後始末

事前に当初のプロファイルに変更を行ってください。

IAMロールを使っている場合、AWS_DEFAULT_PROFILEをunset

```
unset AWS_DEFAULT_PROFILE
```

## 13.1. リモートリポジトリの削除

リポジトリ名の確認

```
cat << ETX

   Repository-Name: ${REPONAME}

ETX
```

```

   Repository-Name: MyDemoRepo

```

リポジトリの削除

```
aws codecommit delete-repository --repository-name ${REPONAME}
```

```
{
    "repositoryId": "********-****-****-****-************"
}
```

確認

```
aws codecommit get-repository --repository-name ${REPONAME}
```

```
A client error (RepositoryDoesNotExistException) occurred when calling the GetRepository operation: MyDemoRepo does not exist
```

## 13.2. ローカルリポジトリの削除

```
cd ~
rm -rf ${REPONAME}
```

## 13.3. 資格情報の削除

設定ファイルを削除

```
rm ~/.ssh/config
```

秘密鍵および公開鍵の名前を確認

```
echo ${SSHKEYNAME}
```

```
id_rsa
```

秘密鍵および公開鍵の名前を削除
(変数が正しく設定されていることを必ず確認してください！！！)

```
rm ~/.ssh/${SSHKEYNAME}
rm ~/.ssh/${SSHKEYNAME}.pub
```

確認

```
ls -al ~/.ssh
```

```
total 16
drwx------ 2 ec2-user ec2-user 4096 Jan  3 10:20 .
drwx------ 3 ec2-user ec2-user 4096 Jan  3 10:19 ..
-rw------- 1 ec2-user ec2-user  391 Jan  3 07:38 authorized_keys
-rw-r--r-- 1 ec2-user ec2-user 1326 Jan  3 08:54 known_hosts
```

## 13.4. IAMユーザの削除

ユーザ名の確認

```
cat << ETX

   iam-user-name: ${GITUSER}

ETX
```

```

   iam-user-name: git-user

```

アタッチされているポリシーの確認

```
ARN=`aws iam list-attached-user-policies --user-name ${GITUSER} --query AttachedPolicies[0].PolicyArn --output text`

echo ${ARN}
```

```
arn:aws:iam::aws:policy/AWSCodeCommitPowerUser
```

マネージドポリシーのデタッチ

```
aws iam detach-user-policy --user-name ${GITUSER} --policy-arn ${ARN}
```

確認

```
aws iam list-attached-user-policies --user-name ${GITUSER}
```

```
{
    "AttachedPolicies": []
}
```

SSH公開鍵IDの確認

```
SSHKEYID=`aws iam list-ssh-public-keys --user-name ${GITUSER} --query SSHPublicKeys[0].SSHPublicKeyId --output text`

echo ${SSHKEYID}
```

```
********************
```

SSH公開鍵の削除

```
aws iam delete-ssh-public-key --user-name ${GITUSER} --ssh-public-key-id ${SSHKEYID}
```

確認

```
aws iam list-ssh-public-keys --user-name ${GITUSER}
```

```
{
    "SSHPublicKeys": [],
    "IsTruncated": false
}
```

アクセスキーの確認

```
ACCESSKEY=`aws iam list-access-keys --user-name ${GITUSER} --query AccessKeyMetadata[0].AccessKeyId --output text`

echo ${ACCESSKEY}
```

```
********************
```

アクセスキーの削除

```
aws iam delete-access-key --user-name ${GITUSER} --access-key-id ${ACCESSKEY}
```

確認

```
aws iam list-access-keys --user-name ${GITUSER}
```

```
{
    "AccessKeyMetadata": []
}
```

ユーザの削除

```
aws iam delete-user --user-name ${GITUSER}
```

確認

```
aws iam get-user --user-name ${GITUSER}
```

```
A client error (NoSuchEntity) occurred when calling the GetUser operation: The user with name git-user cannot be found.
```

以上です。
お疲れ様でした。
