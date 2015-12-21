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

sshキーを作成

```
cd ~
mkdir .ssh
cd .ssh

ssh-keygen -t rsa -b 2048 -f codecommit2
```

```
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in codecommit.
Your public key has been saved in codecommit.pub.
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
total 16
drwxrwxr-x 2 ec2-user ec2-user 4096 Dec 21 09:51 .
drwx------ 8 ec2-user ec2-user 4096 Dec 21 09:51 ..
-rw------- 1 ec2-user ec2-user 1766 Dec 21 09:51 codecommit
-rw-r--r-- 1 ec2-user ec2-user  407 Dec 21 09:51 codecommit.pub
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
aws iam upload-ssh-public-key --user-name ${GITUSER} --ssh-public-key-body file://codecommit.pub
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
  IdentityFile ~/.ssh/codecommit
EOF
```

接続確認

```
sudo ssh git-codecommit.us-east-1.amazonaws.com
```

# ローカルリポジトリの作成

フォルダを作成します

```
cd ~
mkdir temprepo
cd temprepo
```
