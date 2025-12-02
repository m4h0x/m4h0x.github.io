---
title: "NixOS上でのKubernetesクラスタ構築備忘録（sops + niv使用）"
date: 2025-08-31T21:54:20+09:00
draft: false
summary: |
  sops-nixを使ってKubernetes証明書を暗号化転送するまでの流れを，nivによる依存固定とage鍵管理も含めて整理した．
  マスターノードからワーカーノードへ秘密情報を安全に届ける具体的なワークフローを記録している．
categories:
  - Kubernetes
tags:
  - NixOS
  - sops-nix
  - niv
---

<!-- NixOSで構築したKubernetesクラスタに証明書を安全に届けるため，sops-nixを使った暗号化転送パイプラインを検証した備忘録．
nivで依存を固定しつつage鍵を仕込み，マスターからワーカーノードへ機密ファイルを再暗号化しながら配布する手順を整理した．
more -->

## 1. 背景

NixOSでKubernetesクラスタを構築する際，マスターノードからワーカーノードへ証明書を安全に届ける経路が必要だった．`scp`では権限の壁に阻まれて一括送信が難しかったため，`sops-nix`による暗号化転送を試すことにした．`sops-nix`なら秘密鍵をローカルに保持したまま複数ホストへ安全にファイルを配布できる点が魅力である．


## 2. 本題

本環境ではFlakeを使っていない．
Flakeを使わない構成では[niv](https://github.com/nmattia/niv)の利用が推奨されている（[Mic92/sops-nix README](https://github.com/Mic92/sops-nix?tab=readme-ov-file#niv-recommended-if-not-using-flakes)参照）．

### 2-1. nivのインストール準備

`nix-env`で`niv`を取得するにはチャネル登録が必要だったため，[NixOS Wiki](https://nixos.wiki/wiki/Nix_channels)を参考に以下を実行した．

```bash
nix-channel --add https://nixos.org/channels/nixos-25.05 nixos
nix-channel --add https://nixos.org/channels/nixpkgs-25.05-darwin nixpkgs
nix-channel --update
nix-channel --list
```

### 2-2. nivの導入と`sops-nix`追加

```bash
nix-env -iA nixpkgs.niv
niv init
niv add Mic92/sops-nix
```

`configuration.nix`には次のように`sops-nix`モジュールをインポートする．

```nix
{
  imports = [ "${(import ./nix/sources.nix).sops-nix}/modules/sops" ];
}
```

ここまででNixOSのビルドは問題なく通ったため，次は鍵の整備に進む．

### 2-3. age鍵の準備

`age-keygen`が手元に無かったので`ssh-to-age`を用いて鍵を生成した．

```bash
mkdir -p ~/.config/sops/age
nix-shell -p ssh-to-age --run "ssh-to-age -private-key -i ~/.ssh/id_ed25519 > ~/.config/sops/age/keys.txt"
nix-shell -p ssh-to-age --run "ssh-to-age < ~/.ssh/id_ed25519.pub" > ~/.config/sops/age/keys.pub
```

生成した公開鍵をターゲットマシン分そろえておき，後述の`.sops.yaml`で参照する．


### 2-4. sops設定と暗号化フロー

マスターノードに`.sops.yaml`を作成し，ワーカーノードの公開鍵を登録した．

```yaml
keys:
  - &server_worker <公開鍵>
creation_rules:
  - path_regex: /var/lib/kubernetes/secrets/[^/]+\.pem$
    key_groups:
      age:
        - *server_worker
```

### 2-5. PEMファイルの暗号化

PEMファイルをバイナリとして暗号化するため，以下のワンライナーを用意した．

```bash
nix-shell -p sops --run '
for f in /var/lib/kubernetes/secrets/*.pem; do
  sops --input-type binary --output-type binary -e "$f" > "$f.enc"
done
'
```

`sops updatekeys`はYAMLやJSON用でありバイナリには向かないため，再暗号化も`sops -e`で実施する．

```bash
sudo nix-shell -p sops --run '
for f in /var/lib/kubernetes/secrets/*.pem; do
  sops --input-type binary --output-type binary -e "$f" > "$f.enc"
done
'
```

生成した`*.enc`をワーカーノードに転送し，下記のように復号する．

```bash
nix-shell -p sops --run '
for f in *.enc; do
  sops --input-type binary --output-type binary -d "$f" > "${f%.enc}"
done
'
```

### 2-6. ワーカーノードのクラスタ参加

マスターノードの`/home/nix/.kube/config`をコピーし，APIトークンを用いてワーカーノードをクラスタへ参加させた．

```bash
sudo cat /var/lib/kubernetes/secrets/apitoken.secret
# ワーカーノード側
sudo nixos-kubernetes-node-join <apitoken.secret>
```

これで証明書不足による参加失敗を解消できた．

---

## 3. 結論

今回は暗号化ファイルを`scp`で配送して復号するだけの構成だが，`sops-nix`をNixOS設定へ統合すればビルド時に自動復号できるはずである．今後ワーカーノードを増設する際はそちらのワークフローを試す予定だ．
