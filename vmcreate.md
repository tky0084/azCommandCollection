既存のVnetに、VMを複数台作成する。

その後

portalでデータ収集ルールを検索して、新規作成

```
基本情報
規則名：dcr-test
サブスクリプション：subscription-test-defender
リソース グループ：testlog
リージョン：Japan East
テレメトリの種類：Agent-based - Windows or Linux (よく使われる)
データ収集エンドポイント：<none>
リソース：vm-spoke-dev-test-01
仮想マシン：収集と配信
パフォーマンス カウンター：Log Analytics ワークスペース (bvcdvdefdfdf)
タグ
```


```
#========================================
# ① 変数の定義
#========================================

$SubscriptionId = "c6747f3a-f7aa-46ca-afc6-28942c8df653"
$Location = "japaneast"

# 仮想マシンのリソースグループ
$VmRG = "rg-vm-01"

# 既存のネットワーク情報
$VnetRG = "testFirewall"
$VnetName = "vnet0"
$SubnetName = "default"

# 仮想マシンの設定
$VmNamePrefix = "vm-spoke-dev-test-"
$VmCount = 2
$VmSize = "Standard_D2s_v3"

# 管理者アカウント
$Credential = Get-Credential

#========================================
# ② サブスクリプションの切り替え
#========================================

Set-AzContext `
    -SubscriptionId $SubscriptionId `
    -ErrorAction Stop

#========================================
# ③ 既存のVNet・サブネット取得
#========================================

$Vnet = Get-AzVirtualNetwork `
    -ResourceGroupName $VnetRG `
    -Name $VnetName `
    -ErrorAction Stop

$Subnet = Get-AzVirtualNetworkSubnetConfig `
    -VirtualNetwork $Vnet `
    -Name $SubnetName `
    -ErrorAction Stop

#========================================
# ④ リソースグループの作成
#========================================

if (-not (Get-AzResourceGroup -Name $VmRG -ErrorAction SilentlyContinue)) {

    New-AzResourceGroup `
        -Name $VmRG `
        -Location $Location `
        -ErrorAction Stop
}

#========================================
# ⑤ 仮想マシンの作成
#========================================

for ($i = 1; $i -le $VmCount; $i++) {

    $VmName = "{0}{1:D2}" -f $VmNamePrefix, $i
    $NicName = "$VmName-nic"
    $ComputerName = "vm-test-{0:D2}" -f $i

    # 既存NICの確認
    $Nic = Get-AzNetworkInterface `
        -Name $NicName `
        -ResourceGroupName $VmRG `
        -ErrorAction SilentlyContinue

    # NICが存在しない場合は新規作成
    if (-not $Nic) {

        $Nic = New-AzNetworkInterface `
            -Name $NicName `
            -ResourceGroupName $VmRG `
            -Location $Location `
            -SubnetId $Subnet.Id `
            -ErrorAction Stop
    }

    # 仮想マシンの構成定義
    $VmConfig = New-AzVMConfig `
        -VMName $VmName `
        -VMSize $VmSize

    # OSの設定
    $VmConfig = Set-AzVMOperatingSystem `
        -VM $VmConfig `
        -Windows `
        -ComputerName $ComputerName `
        -Credential $Credential `
        -ProvisionVMAgent `
        -EnableAutoUpdate

    # Windows Server 2022 Azure Edition
    $VmConfig = Set-AzVMSourceImage `
        -VM $VmConfig `
        -PublisherName "MicrosoftWindowsServer" `
        -Offer "WindowsServer" `
        -Skus "2022-datacenter-azure-edition" `
        -Version "latest"

    # NICの関連付け
    $VmConfig = Add-AzVMNetworkInterface `
        -VM $VmConfig `
        -Id $Nic.Id

    # ブート診断の無効化
    $VmConfig = Set-AzVMBootDiagnostic `
        -VM $VmConfig `
        -Disable

    # 仮想マシンの作成
    New-AzVM `
        -ResourceGroupName $VmRG `
        -Location $Location `
        -VM $VmConfig `
        -ErrorAction Stop

    Write-Host "仮想マシンの作成完了：$VmName"
}
```
