ひと通り書けたと思います。

# 前提条件

## 権限

ハンズオンは以下の権限を付与されたユーザ（もしくはロール）で実行してください。

- CodeCommitに関するフルコントロール権限
- IAMに関するフルコントロール権限

# 事前準備

リージョンを設定します。
現時点（2015年12月21日）では、バージニアのみサポート

```
export AWS_DEFAULT_REGION='us-east-1'
```

# リポジトリの作成

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

# 認証情報の設定

## キーペアの作成

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
（パスワードの設定は任意）

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

## IAMユーザの作成

ユーザ名を設定します

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

## 公開鍵をアップロード

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

## 認証情報の設定（SSH接続の場合）

公開鍵のIDを取得

```
SSHPUBID=`aws iam list-ssh-public-keys --user-name ${GITUSER} | jq -r '.SSHPublicKeys[0].SSHPublicKeyId'`

cat << ETX

   SSHPublicKeyId: ${SSHPUBID}

ETX
```

```

   SSHPublicKeyId: ********************

```

設定ファイルを作成（既存ファイルがある場合はリネームしておくなどの対策を！）

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

接続確認

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

## 権限を付与

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

# Gitをインストール

インストール

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

# ユーザ名およびメールアドレスの設定

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
git config --global user.email "${MYEMAIL}"
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

# ローカルリポジトリの作成

## ディレクトリの作成

ディレクトリの作成

```
cd ~
mkdir ${REPONAME}
cd ${REPONAME}
```

## 初期化

```
git init
```

```
Initialized empty Git repository in /home/ec2-user/MyDemoRepo/.git/
```

# インデックスの状態を確認

```
git status
```

```
On branch master

Initial commit

nothing to commit (create/copy files and use "git add" to track)
```

# マスターブランチの変更

## コンテンツを追加

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

## 変更をコミット

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

# 変更をプッシュ

プッシュ前の状態を確認（リポジトリは空の状態）

```
aws codecommit get-branch --repository-name ${REPONAME} --branch-name  master
```

```
A client error (BranchDoesNotExistException) occurred when calling the GetBranch operation: None
```

URL(SSH)を確認

```
SSHURL=`aws codecommit get-repository --repository-name MyDemoRepo | jq -r .repositoryMetadata.cloneUrlSsh`

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

# ブランチの作成、コンテンツの変更

## ブランチの作成

コミットIDの確認

```
COMMITID=`aws codecommit get-branch --repository-name ${REPONAME} --branch-name master | jq  -r .branch.commitId`
```

ブランチ名の決定

```
BRANCHNAME="MyNewBranch"
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

リモートリポジトリからローカルリポジトリへ新しいブランチのコンテンツをフェッチ

```
git fetch origin ${BRANCHNAME}
```

```
Warning: Permanently added the RSA host key for IP address '54.239.20.180' to the list of known hosts.
From ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
 * branch            MyNewBranch -> FETCH_HEAD
 * [new branch]      MyNewBranch -> origin/MyNewBranch
```


ブランチのスイッチ

```
git checkout ${BRANCHNAME}
```

```
Branch MyNewBranch set up to track remote branch MyNewBranch from origin.
Switched to a new branch 'MyNewBranch'
```

確認1

```
git branch
```

```
* MyNewBranch
  master
```

確認2

```
git status
```

```
On branch MyNewBranch
Your branch is up-to-date with 'origin/MyNewBranch'.
nothing to commit, working directory clean
```

## コンテンツの修正

```
cat << EOF >> ${FILENAME}
write for new branch
EOF

cat ${FILENAME}
```

```
write for new branch
```

追加

```
git add ${FILENAME}
```

コミット

```
git commit -m "add new line"
```

```
[MyNewBranch 3e121a8] add new line
 1 file changed, 1 insertion(+)
```

## マスターブランチへのマージ

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

## リモートリポジトリへのプッシュ

```
git push origin --all
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
        "branchName": "MyNewBranch"
    }
}
```

# オプション

## clone

後で書く

```
cd ~

git clone ${SSHURL} ${REPONAME}

cd ${REPONAME}
```

## revert

後で書く（そもそも可能？）

## reset

後で書く（そもそも可能？）

## rebase

後で書く（そもそも可能？）

## タグの利用

後で書く

## 競合の解決

後で書く


# 後始末

## リモートリポジトリの削除

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

## ローカルリポジトリの削除

```
cd ~
rm -rf ${REPONAME}
```

## 資格情報の削除

設定ファイルを確認

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
(変数が設定されていることを必ず確認してください！！！)

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

## IAMユーザの削除

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
ARN=`aws iam list-attached-user-policies --user-name ${GITUSER} | jq -r .AttachedPolicies[].PolicyArn`

echo ${ARN}
```

```
arn:aws:iam::aws:policy/AWSCodeCommitPowerUser
```

マネージドポリシーの削除

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
SSHKEYID=`aws iam list-ssh-public-keys --user-name ${GITUSER} | jq -r .SSHPublicKeys[].SSHPublicKeyId`

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
