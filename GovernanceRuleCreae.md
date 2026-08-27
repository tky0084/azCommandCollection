```
# ================================================================
# モジュールのインポート
# ================================================================
import argparse # コマンドライン引数を受け取るために必要なモジュール
import json # PythonのデータとJSONを変換する
import subprocess # PythonからAzure CLIなどの外部コマンドを実行する
import uuid # ガバナンスルール用の一意なIDを生成する
import sys # エラー出力やスクリプト終了を制御する

# ================================================================
# Azure CLIを実行する関数(メタデータに関する設定を維持するため関数化する)
# ================================================================
def run_az_command(command):
    """
    Azure CLIコマンドを実行する関数

    command:
        ["account", "show", ...] のようなリストを指定
    """
	
	# コマンドの実行し、その結果に関する情報をresultに格納
    result = subprocess.run(
        ["az"] + command,
        capture_output=True, # コマンドの実行結果をPythonで受け取る
        text=True, # コマンドの出力を文字列として扱う指定
        encoding="utf-8" # 文字列をUTF-8として扱う
    )

    # Azure CLIの実行に失敗した場合
    if result.returncode != 0:
        raise RuntimeError(
            f"Azure CLIの実行に失敗しました。\n"
            f"Command: az {' '.join(command)}\n"
            f"Error: {result.stderr}"
        )

    return result.stdout


# ================================================================
# 引数(サブスクリプション名と通知先メールアドレス)
# ================================================================
# 引数を受け取るための仕組み」を作成
parser = argparse.ArgumentParser(
    description="Defender for Cloud ガバナンスルール作成"
)

# 引数subscription_nameを定義
parser.add_argument(
    "--subscription_name",
    required=True,
    help="対象サブスクリプション名"
)

# 引数mail-addressを定義
parser.add_argument(
    "--mail-address",
    required=True,
    help="ガバナンスルールのOwnerメールアドレス"
)

# 実際に入力された値を読み取る
args = parser.parse_args()

# 読み取った値から、各パラメータを変数に定義
subscription_name = args.subscription_name.strip()
mail_address = args.mail_address.strip()


# ================================================================
# 入力チェック
# ================================================================
if not subscription_name:
    raise ValueError(
        "subscription_name が指定されていません。"
    )

if not mail_address:
    raise ValueError(
        "MailAddress が指定されていません。"
    )

try:
    # ================================================================
    # 対象サブスクリプション設定
    # ================================================================
    # サブスクリプション移動のログ
    print(
        f"サブスクリプションを設定しています: "
        f"{subscription_name}"
    )
	
	# Az CLIコマンド"az account set --subscription {subscription_id}"のコマンドを実行
	# サブスクリプションの移動
    run_az_command([
        "account",
        "set",
        "--subscription",
        subscription_name
    ])


    # ================================================================
    # サブスクリプション名取得
    # ================================================================
   	# Az CLIコマンド"az account show --subscription {subscription_id} --query name --output tsv"のコマンドを実行
    subscription_id = run_az_command([
        "account",
        "show",
        "--subscription",
        subscription_name,
        "--query",
        "id",
        "--output",
        "tsv"
    ]).strip()

	# サブスクリプションidを出力
    print(
        f"サブスクリプションid: "
        f"{subscription_id}"
    )


    # ================================================================
    # Governance Rule ID作成
    # ================================================================
    # ルールのIDをGUID形式で生成
    rule_id = str(uuid.uuid4())

    print(
        f"Governance Rule ID: "
        f"{rule_id}"
    )


    # ================================================================
    # REST API URL
    # ================================================================
    # Python構文文字列フォーマット文字列リテラルを使用して、文字列の中に変数を埋め込む
    # 改行区切りで可読性を向上
    uri = (
        f"https://management.azure.com/"
        f"subscriptions/{subscription_id}/"
        f"providers/Microsoft.Security/"
        f"governanceRules/{rule_id}"
        f"?api-version=2022-01-01-preview"
    )


    # ================================================================
    # リクエストBody
    # ================================================================
    body = {
        "properties": {
            "displayName":
                f"goverules_{subscription_name}",

            "description":
                f"{subscription_name}のガバナンスルール",

            "isDisabled": False,

            "ruleType": "Integrated",

            "rulePriority": 1,

            "remediationTimeframe":
                "90.00:00:00",

            "isGracePeriod": False,

            "ownerSource": {
                "type": "Manually",
                "ownerType": "User",
                "value": mail_address
            },

            "conditionSets": [
                {
                    "conditions": [
                        {
                            "property":
                                "properties.risk.level",

                            "value":
                                "[\"Critical\",\"High\"]",

                            "operator":
                                "In"
                        }
                    ]
                }
            ],

            "sourceResourceType":
                "Assessments",

            "includeMemberScopes":
                False,

            "governanceEmailNotification": {
                "disableManagerEmailNotification":
                    True,

                "disableOwnerEmailNotification":
                    False,

                "emailNotificationDayOfWeek":
                    "Monday"
            }
        }
    }


    # ================================================================
    # Governance Rule 作成
    # ================================================================
    print(
        f"送信中: {uri}"
    )

	# REST API用データを扱うため、JSONに変換
    body_json = json.dumps(body)

	# az CLIでREST APIの実行
    response = run_az_command([
        "rest",
        "--method",
        "put",
        "--uri",
        uri,
        "--body",
        body_json,
        "--headers",
        "Content-Type=application/json"
    ])


    # ================================================================
    # 結果表示
    # ================================================================

	# response の前後にある空白や改行を削除し、その結果に文字が残っていれば True、空文字 "" なら False として扱う
    if response.strip():

        response_json = json.loads(response)

        print(
            json.dumps(
                response_json,
                indent=2,
                ensure_ascii=False
            )
        )

    print(
        "ガバナンスルールの作成に成功しました。"
    )


except Exception as e:

    print(
        f"処理に失敗しました: {e}",
        file=sys.stderr
    )

    sys.exit(1)
```
