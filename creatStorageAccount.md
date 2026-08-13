# ストレージアカウントの作成

```
# PowerShellでAzureアカウントにログインする
Connect-AzAccount

# 変数定義
$strageName = "XXXXXXXX" #ストレージアカウント名
$resourceGroup = "XXXXXXXX" #ストレージアカウント名
$location = "japaneast" #ストレージアカウント名

# リソースグループ作成
New-AzResourceGroup -Name $resourceGroup -Location $location

# ストレージアカウント作成
New-AzStorageAccount -ResourceGroupName $resourceGroup `
  -Name $strageName `
  -Location $location `
  -SkuName Standard_LRS `
  -Kind StorageV2
```