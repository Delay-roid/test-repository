# ポートフォリオ①：エンジン実験データ生成ツール 設計書

## 1. 概要

### 何を作るか
エンジン実験の時系列データを生成するVBAツール

### 何のために
- 実験データがないため、VBAで疑似的な時系列データを作成する
- 燃焼割合50%時点の温度・圧力を抽出する
- サイクル平均データに統合する

### 作業の流れ
```
実験条件入力（Excel設定シート）
  ↓
VBAで時系列データ計算
  ↓
CSVファイル出力
  ↓
SQLiteデータベースに保存
  ↓
GitHubにアップロード
```

### 成果物
- `timeseries.csv` - 時系列データ（クランク角度ごとの詳細データ）
- `cycle_average.csv` - サイクル平均データ（運転条件ごとのまとめ）
- `data_generator.xlsm` - データ生成VBAツール
- `engine_data.sqlite` - SQLiteデータベース

---

## 2. データの流れ（全体像）
```
[1. Excelの設定シート]
   ├─ 運転条件パラメータ
   ├─ 計算パラメータ
   └─ クランク角度範囲
     ↓
[2. VBAで計算] ← 詳細は「3. 詳細設計」
   ├─ 燃焼割合計算（Wiebe関数）
   ├─ 温度計算
   └─ 圧力計算
     ↓
[3. 配列に格納]
   └─ メモリ上で高速処理
     ↓
[4. CSVファイル出力]
   ├─ timeseries.csv
   └─ cycle_average.csv
     ↓
[5. SQLiteに保存] ← 詳細は「4. データベース設計」
   ├─ cycle_averageテーブル
   └─ timeseriesテーブル
     ↓
[6. GitHubにアップロード]
```

---

## 3. 詳細設計（2番のVBA計算部分）

### 3.1 入力パラメータ

#### 運転条件（3パターン）

| 運転条件ID | 回転数(rpm) | トルク(Nm) |
|-----------|------------|-----------|
| RUN001    | 2000       | 150       |
| RUN002    | 2500       | 200       |
| RUN003    | 3000       | 180       |

#### クランク角度範囲
- 開始: -10°
- 終了: 20°
- 刻み幅: 1°
- データ点数: 31点/条件

### 3.2 計算式とパラメータ

#### 燃焼割合の計算（Wiebe関数）

**式:**
```
MFB(θ) = 1 - exp(-a * ((θ - θ0) / Δθ)^(m+1))
```

**パラメータ（運転条件ごと）:**

| 運転条件ID | θ0(°) | Δθ(°) | a | m |
|-----------|-------|-------|---|---|
| RUN001    | -10   | 30    | 5 | 2 |
| RUN002    | -10   | 25    | 5 | 2 |
| RUN003    | -10   | 20    | 5 | 2 |

**変数の意味:**
- θ: クランク角度
- θ0: 燃焼開始角度
- Δθ: 燃焼期間
- a: 燃焼効率パラメータ
- m: 形状パラメータ（前後期比）
- MFB: 燃焼割合（0～1）

#### 温度の計算

**式:**
```
T(MFB) = T_init + (T_peak - T_init) * MFB^n
```

**パラメータ:**
- T_init = 600K（初期温度）
- T_peak = 1500K（最高温度）
- n = 1.3（形状パラメータ）

#### 圧力の計算

**式:**
```
P(MFB) = P_init + (P_peak - P_init) * MFB^m
```

**パラメータ:**
- P_init = 2.0MPa（初期圧力）
- P_peak = 6.0MPa（最高圧力）
- m = 1.2（形状パラメータ）

### 3.3 VBA処理フロー
```
Sub GenerateData()
    ' 設定シートから値を読み込み
    
    For 各運転条件 (RUN001～RUN003)
        ' 運転条件のパラメータ取得
        
        For クランク角度 = -10 to 20 (step 1)
            ' ステップ1: 燃焼割合を計算
            MFB = CalcMFB(angle, θ0, Δθ, a, m)
            
            ' ステップ2: 温度を計算
            temp = CalcTemperature(MFB)
            
            ' ステップ3: 圧力を計算
            press = CalcPressure(MFB)
            
            ' 配列に格納
            data(i, 1) = 運転条件ID
            data(i, 2) = angle
            data(i, 3) = MFB
            data(i, 4) = temp
            data(i, 5) = press
        Next
    Next
    
    ' CSVファイルに書き出し
    Call WriteToCSV(data)
End Sub
```

---

## 4. データベース設計（5番のSQLite部分）

### 4.1 テーブル構造

#### テーブル1: cycle_average（サイクル平均データ）
```sql
CREATE TABLE cycle_average (
    run_id TEXT PRIMARY KEY,
    rpm INTEGER NOT NULL,
    torque REAL NOT NULL
);
```

**データ例:**
| run_id | rpm  | torque |
|--------|------|--------|
| RUN001 | 2000 | 150    |
| RUN002 | 2500 | 200    |
| RUN003 | 3000 | 180    |

#### テーブル2: timeseries（時系列データ）
```sql
CREATE TABLE timeseries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    crank_angle REAL NOT NULL,
    mfb REAL NOT NULL,
    temperature REAL NOT NULL,
    pressure REAL NOT NULL,
    FOREIGN KEY (run_id) REFERENCES cycle_average(run_id)
);
```

**データ例:**
| id | run_id | crank_angle | mfb  | temperature | pressure |
|----|--------|-------------|------|-------------|----------|
| 1  | RUN001 | -10.0       | 0.00 | 600.0       | 2.0      |
| 2  | RUN001 | -9.0        | 0.05 | 650.0       | 2.3      |
| 3  | RUN001 | -8.0        | 0.12 | 700.0       | 2.8      |
| ...| ...    | ...         | ...  | ...         | ...      |

### 4.2 データ投入方法

1. VBAでCSVファイルを生成
2. DBeaverでSQLiteに接続
3. CSVファイルをインポート

---

## 5. 前提条件・制約事項

### 前提条件
- 時系列データに燃焼割合50%のデータ点が存在する前提
- データ補間は考慮しない（将来の改修で対応）

### 制約事項
- 圧力・温度は燃焼終了時に最大となる（実際とは異なる簡易モデル）
- 膨張による圧力・温度低下は未実装

### 将来の改修予定（12月-1月）
- 燃焼割合50%に対する線形補間の実装
- より現実的なパラメータ範囲の設定
- 膨張過程の圧力・温度計算
- エラーハンドリングの追加