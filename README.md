# TouchMap
<img width="3814" height="1860" alt="image" src="https://github.com/user-attachments/assets/660acc3a-6b8f-429a-a34f-7ec2da1fa92d" />

VR ハンドインタラクション用の GPU 接触ヒートマップシステムです。  
手がオブジェクトに触れた部分をリアルタイムで色の変化として可視化し、触覚デバイスへのデータ出力にも対応しています。

---

## セットアップ

### ① 触れるオブジェクトの設定

触覚反応の対象にしたい各オブジェクトに **`TouchMapBaker`** コンポーネントを追加してください。

- メッシュに UV がある場合 → GPU ベイク（高精度）
- メッシュに UV がない場合 → 頂点フォールバック（自動切替）

### ② マネージャーの設定

シーンに空の GameObject を作り、**`TouchMapManager`** を追加します。

Inspector で以下を設定してください：

| フィールド | 説明 |
|---|---|
| `targetObjects` | `TouchMapBaker` を付けたオブジェクトをすべてドラッグ |
| `hands` | 手の Transform を配列で指定（例: `[0]=左手`, `[1]=右手`） |
| `touchMapCompute` | `TouchMapCompute.compute` をドラッグ |
| `contactThreshold` | 接触検知の距離（メートル単位、デフォルト: 0.05） |
| `heatmapResolution` | ヒートマップの解像度（デフォルト: 256） |

### ③ 手の設定

`hands` に指定する Transform の子に **`SkinnedMeshRenderer`** が必要です。

---

## ⚠️ 注意事項（必ず確認してください）

### メッシュの Read/Write

**すべてのメッシュで Read/Write を ON にしてください。**

- 手のメッシュ（SkinnedMeshRenderer）
- 触れるオブジェクトのメッシュ（MeshFilter）

設定方法：
1. Project ウィンドウでメッシュのモデルファイル（.fbx 等）を選択
2. Inspector → Model タブ
3. **「Read/Write Enabled」にチェック**
4. Apply

Read/Write が OFF のメッシュは TouchMap で使用できません。

### シェーダーがピンク色になる場合

マテリアルがピンク色になった場合、Console ウィンドウにエラー内容が表示されているはずです。確認してください。

よくある原因：
- 対象オブジェクトのメッシュに UV がない
- 手のメッシュや参照が null になっている
- `touchMapCompute` や `hands` が Inspector で未設定

### Console の [TouchMap] ログについて

TouchMap は初期化時に `[TouchMap]` プレフィックス付きのログを出力します。  
問題が起きた場合は、まず Console で `[TouchMap]` を検索してください。

| ログの種類 | 意味 |
|---|---|
| `'オブジェクト名' produced 0 points` | メッシュからデータを取得できなかった（Read/Write を確認） |
| `Mesh has no UVs. Using vertex fallback` | UV がないため頂点直接サンプリングに切り替えた（正常動作） |
| `Mesh has Read/Write DISABLED` | Read/Write が OFF です。インポート設定を変更してください |
| `No SkinnedMeshRenderer on hand` | 手に SkinnedMeshRenderer が見つからない |
| `Extracted ○○ points` | 正常にポイントクラウドが生成された |

---

## ファイル構成

| ファイル | 役割 |
|---|---|
| `TouchMapManager.cs` | メイン制御。GPU パイプラインの管理、両手対応 |
| `TouchMapBaker.cs` | オブジェクトのメッシュをポイントクラウドに変換 |
| `TouchMapCompute.compute` | コンピュートシェーダー。UV 空間で接触計算を実行 |
| `TouchMapHeatmap.shader` | 手に表示されるヒートマップのシェーダー |
| `TouchMapBake.shader` | メッシュを UV 空間に展開する内部用シェーダー |

---

## ヒートマップの色

手に表示される色の意味：

| 色 | 意味 |
|---|---|
| グレー | 接触なし |
| 青 → 緑 | 軽い接近 |
| 緑 → 黄 | 中程度の接触 |
| 黄 → 赤 | 強い接触・めり込み |

---

## スクリプトからの利用（API）

```csharp
using TouchMap;

TouchMapManager manager = FindObjectOfType<TouchMapManager>();
```

### 1点クエリ

```csharp
// 指定した UV 座標のヒート値を取得
// handIndex: 0=左手, 1=右手
// 戻り値: 0.0（接触なし）〜 1.0（最大接触）
float heat = manager.GetHeatAtUV(0, new Vector2(0.5f, 0.5f));
```

### 矩形範囲の平均ヒート

```csharp
// uvMin〜uvMax の矩形範囲内の平均ヒート値を取得
// ハプティックデバイスへの出力に最適
float avg = manager.GetHeatInRect(0, new Vector2(0.2f, 0.3f), new Vector2(0.8f, 0.7f));
```

### 矩形範囲の最大ヒート

```csharp
// uvMin〜uvMax の矩形範囲内で最も強いヒート値を取得
float max = manager.GetMaxHeatInRect(0, new Vector2(0.4f, 0.9f), new Vector2(0.6f, 1.0f));
```

### ハプティックデバイスへの送信例

```csharp
// 自分で好きな UV 範囲を定義
Vector2 palmMin = new Vector2(0.2f, 0.1f);
Vector2 palmMax = new Vector2(0.8f, 0.5f);

// 左手の手のひら領域の平均ヒートを取得
float palmHeat = manager.GetHeatInRect(0, palmMin, palmMax);

// デバイスに送信（実装はデバイスに依存）
hapticDevice.SetIntensity(palmHeat);
```

> **UV 範囲の定義はユーザー側で行います。**  
> 手のメッシュの UV マップを確認し、必要な領域の座標を指定してください。

---

## 動作確認環境

| 項目 | 内容 |
|---|---|
| GPU | NVIDIA GeForce RTX 4060 |
| メモリ | 16 GB |
| VR SDK | Meta XR SDK |
| Unity | 2021.3+ / URP |
| Compute Shader | Shader Model 5.0 |
