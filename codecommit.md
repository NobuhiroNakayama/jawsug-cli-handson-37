まだまだ書きかけです。。。

# 事前準備

リージョンを設定します。
現時点（2015年12月21日）では、バージニアのみサポート

```
export AWS_DEFAULT_REGION='us-east-1'
```

# リポジトリの作成

リポジトリ名および説明を設定します
（リポジトリ名はグローバルでユニークである必要はあるのか？？？）

```
REPONAME='MyDemoRepo'
REPODESC='My demonstration repository'
```

変数を確認します

```
cat << ETX

   repository-name: '${REPONAME}'
   repository-description: '${REPODESC}'

ETX
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
            "repositoryId": "6a43da82-fa01-4bf3-b2ed-ca4f7fe179b1"
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

sshキーの名前を設定

```
SSHKEYNAME='id_rsa'
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
**:**:**:**:**:**:**:**:**:**:**:**:**:**:**:** ec2-user@ip-172-31-13-173
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

ユーザ名を確認します

```
cat << ETX

   repository-name: '${GITUSER}'

ETX
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

公開鍵をアップロードします。

```
aws iam upload-ssh-public-key --user-name ${GITUSER} --ssh-public-key-body file://${SSHKEYNAME}
```

```
{
    "SSHPublicKey": {
        "UserName": "git-user",
        "Status": "Active",
        "SSHPublicKeyBody": "ssh-rsa **************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************** ec2-user@ip-172-31-13-173\n",
        "UploadDate": "2015-12-21T10:10:54.908Z",
        "Fingerprint": "**:**:**:**:**:**:**:**:**:**:**:**:**:**:**:**",
        "SSHPublicKeyId": "********************"
    }
}
```

公開鍵のIDを取得

```
SSHPUBID=`aws iam list-ssh-public-keys --user-name ${GITUSER} | jq .SSHPublicKeys | jq -r .[].SSHPublicKeyId`

echo ${SSHPUBID}
```

認証情報の設定

```
cat << EOF > ~/.ssh/config
Host git-codecommit.*.amazonaws.com
  User ${SSHPUBID}
  IdentityFile ~/.ssh/${SSHKEYNAME}
EOF
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

```
You have successfully authenticated over SSH. You can use Git to interact with AWS CodeCommit. Interactive shells are not supported.Connection to git-codecommit.us-east-1.amazonaws.com closed by remote host.
Connection to git-codecommit.us-east-1.amazonaws.com closed.
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
MYEMAIL=user01@example.com
```

ユーザ名とメールアドレスを設定

```
git config --global user.name "${MYNAME}"
git config --global user.email "${MYEMAIL}"
```

# リポジトリのクローン

URL(SSH)を確認

```
SSHURL=`aws codecommit get-repository --repository-name MyDemoRepo | jq -r .repositoryMetadata.cloneUrlSsh`

echo ${SSHURL}
```

リポジトリをクローン

```
cd ~

git clone ${SSHURL} ${REPONAME}

cd ${REPONAME}
```

# インデックスの状態を確認

```
git status
```

# リポジトリの変更

ファイルを作成

```
touch codecommit.txt
```

ファイルを追加

```
git add codecommit.txt
```

# 変更をコミット

コミット

```
git commit -m "touch codecommit.txt"
```

```
[master (root-commit) cd4123c] First Commit
 Committer: EC2 Default User <ec2-user@ip-172-31-13-173.ap-northeast-1.compute.internal>
Your name and email address were configured automatically based
on your username and hostname. Please check that they are accurate.
You can suppress this message by setting them explicitly. Run the
following command and follow the instructions in your editor to edit
your configuration file:

    git config --global --edit

After doing this, you may fix the identity used for this commit with:

    git commit --amend --reset-author

 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 codecommit.txt
```

確認

```
git log
```

# 変更をプッシュ

コミットしたコンテンツをプッシュ

```
git push origin master
```

```
Counting objects: 3, done.
Writing objects: 100% (3/3), 249 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
To ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
 * [new branch]      master -> master
```

# ブランチの作成、コンテンツの変更

## コミットIDの確認

```
COMMITID=`aws codecommit get-branch --repository-name ${REPONAME} --branch-name master | jq  -r .branch.commitId`
echo ${COMMITID}
```

## ブランチの作成

```
BRANCHNAME="MyNewBranch"
```

```
aws codecommit create-branch --repository-name ${REPONAME} --branch-name ${BRANCHNAME} --commit-id ${COMMITID}
```



## ブランチの変更

ブランチのスイッチ

```
git checkout ${BRANCHNAME}
```

確認

```
git branch
git status
```

## マスターブランチへのマージ

```
git checkout master
git branch
```

```
git merge ${BRANCHNAME}
```

## リモートリポジトリへのプッシュ

```
git push origin
```

# 後始末

## リモートリポジトリの削除

```
aws codecommit delete-repository --repository-name ${REPONAME}
```

```
{
    "repositoryId": "********-****-****-****-************"
}
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

秘密鍵および公開鍵の名前を削除
(変数が設定されていることを必ず確認してください！！！)

```
rm ~/.ssh/${SSHKEYNAME}*
```

## IAMユーザの削除

```
aws iam delete-user --user-name ${GITUSER}
```
