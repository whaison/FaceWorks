
NVIDIA FaceWorks 1.0
==================

目次
-----------------

* [概要]（＃概要）
* [GitHubリリースノート]（＃github-release-notes）
* [リポジトリの内容]（＃リポジトリの内容）
* [機能概要]（＃機能概要）
* [統合ガイド]（＃統合ガイド）
* [バージョン履歴]（＃バージョン履歴）

概要
--------

FaceWorksは、NVIDIAの[「Digital Ira」デモ]（http://www.nvidia.com/coolstuff/demos#!/lifelike-human-face）で使用されている技術に触発された、高品質のスキンとアイレンダリング用のミドルウェアライブラリです。 -レンダリング）。

Digital Iraは、USCのDr. Paul Debevecのチームと協力してNVIDIAによって作成されたハイテクデモです。高解像度の頭部スキャンとパフォーマンスキャプチャが、リアルタイムで可能な限り現実的にレンダリングされます。デモチームは、考えることができる。対照的に、FaceWorksは、再利用可能で、プロダクション対応のライブラリであり、ゲーム開発者がDigital Iraのレンダリング品質に合わせることができるようにすることを目指しています。

FaceWorksは現在、2つの主な機能を提供しています：

- 直接光と周囲光の両方をサポートする、高品質で効率的な地下散乱ソリューション。
- シャドーマップからの厚さの推定に基づいて、直接光に対する深い散乱（半透明）の実装。

現在、FaceWorksがサポートする唯一のAPIはDirect3D 11です。

GitHubリリースノート
--------------------

このリポジトリはgitサブモジュールとしてDXUTを使用します。プロジェクトを正しくビルドするには、メインリポジトリディレクトリでgitシェルを開き、次のように入力してサブモジュールを初期化する必要があります。

`git submodule update --init`

リポジトリの内容
--------------------------

- `doc /` FaceWorksに関するドキュメンテーションの画像とスライド。
- `include /`プロジェクトにインクルードするC / C ++とHLSLヘッダファイル。
- facesWorksの使い方を示すサンプルアプリケーションのための `samples /`ソース。
     - VS 2012、2013および2015用の `samples / build /`プロジェクト/ソリューションファイル。それぞれに32ビットと64ビットのビルドがあります。
     - サンプルアプリケーションで使用される `samples / externals / 'サードパーティコード。
     - `samples / d3d11 /`対話式D3D11サンプル。
     - サブサーフェス散布アルゴリズムで使用されるルックアップテクスチャ（LUT）を構築するための `samples / lut_generator /`コマンドラインユーティリティ。
     - インタラクティブサンプルで使用される `samples / media /`モデルとテクスチャ。
- `src /` C / C ++とHLSLソースファイル、そしてVS 2012、2013、2015プロジェクトファイルを使用して、それぞれ32ビットと64ビットのビルドライブラリをビルドします。

機能の概要
----------------

このセクションでは、FaceWorksのレンダリングがどのように動作し、どのデータがどのように使用されるかについての概要を説明します。 FaceWorksをゲームエンジンに統合する方法については、統合ガイドを参照してください。

また、FaceWorksの概要については、[pdf]（doc / slides / FaceWorks-Overview-GTC14.pdf）および[pptx]（doc / slides / FaceWorks-Overview-GTC14.pptx）形式を使用します。

###地下散乱

！[人間の顔の表面下散乱の有無の比較（[拡大版]（doc / images / sss_comparison_large.png））]（doc / images / sss_comparison.png）

表面下散乱（SSS）は、光が皮膚に浸透し、異なる点で跳ね返り、そこを出る物理的過程です。上のスクリーンショットに見られるように、それがなければ、皮膚が不自然に硬く乾燥しているように見えるので、現実的な人間の皮膚の外観の重要な部分です。

FaceWorksには、Eric Pennerの2010 SIGGRAPH論文[Pre-Integrated Skin Shading]（http://advances.realtimerendering.com/s2011/Penner%20-%20Pre-Integrated%20Skin%20Rendering%20%28Siggraph）に基づくSSSの実装が含まれています％202011％20Advances％20in％20リアルタイム％20レンダリング％20Course％29.pptx）。これには、テクスチャ空間拡散などのマルチパスメソッドよりも効率的になり、ゲームエンジンに統合しやすくなる単一のレンダリングパスだけが必要であるという利点があります。

さらに、FaceWorksは周囲光と直接光の両方のSSSをサポートしています。周囲光は、球面調和関数、画像ベースの照明、または他の方法を使用して実装できます。唯一の要件は、ピクセルシェーダーのさまざまな法線ベクトルに対して評価できることです。

SSSの実装は、次の4つのコンポーネントに分かれています。

**曲率：** FaceWorksレンダリングが適用される各メッシュの頂点ごとの曲率値を事前計算します。 SSSは、より大きな曲率を有するメッシュの領域においてより明白である。ピクセルシェーダでは、各点における曲率およびN・Lを使用して、事前計算された散乱結果を格納するルックアップテクスチャ（LUT）をサンプリングする。

**ノーマルマップ<！--- - >：**ノーマルマップを通常通り2回、ミップレベルを高くして1回サンプリングし、SSS半径に対応するぼやけた法線を取得します。両方の法線からの照明は曲率項と組み合わされて全拡散照明に到達する。正しいミップレベルを計算するために、FaceWorksはまた、メッシュ毎に事前計算するUVスケールを知る必要があります。
-----------------
**シャドウエッジ：**ライトは、ライトエリアからシャドウに拡散し、シャドウエッジの内側に赤色の輝きを作ります。これには、シャドウフィルタを広げ、シャドウをシャープにし、残りの値の範囲を使用してその赤い輝きを追加する方法があります。シャドーフィルタリングの詳細はあなたに任せます。一貫したフィルタ半径（光源から見て）を与える限り、どのような方法でも使用できます。接触硬化法はここでは望ましくありません。シャドウフィルタの値が広いと、FaceWorksはそれを使って事前計算された散乱結果を格納している別のLUTをサンプリングし、最終的なシャドーカラーを与えます。影のエッジを横切るときにシャドウ値が0から1に変化する）は、SSS拡散フィルタと少なくとも同じ幅でなければならない。さまざまなFaceWorksコンフィグレーション構造体に設定されている半径は、拡散プロファイルの最も広いガウス分布のシグマであるため、合計フィルタ幅は約6倍です。人間の皮膚の場合、拡散半径は2.7mmであるため、シャドーフィルタの幅は理想的には少なくとも16.2mmでなければなりません。しかし、このような大きなシャドーフィルタは、性能と美的理由の両方で実用的ではありません。幅の狭いフィルタを使用すると、シャドーエッジを越えて光がブリードすることはありませんが、FaceWorksで計算されたSSSの結果は滑らかな減衰となり、目に見えるように見えます。以前は、複数の法線ベクトルを評価できる限り、任意のタイプのアンビエント照明を使用できます。 FaceWorksは、前に見た2つの法線ベクトルと3つ目の中間法線を使用します。周囲光がこれらの法線のそれぞれについて評価され、次に3つの照明値が1つに戻って組み合わされます.FaceWorksは、これらの各成分による照明を評価します。その結果を通常の拡散照明用語のように拡散色と明るい色で乗算し、選択した鏡面モデルを追加することができます### Deep Scatter！[人の顔を深い散乱の有無に応じて比較するSSSは、光の拡散を皮膚の表面に沿って局所的にモデル化しますが、光は、モデルの大部分を介して散乱することもあります。このモデルでは、次のような問題が発生しています。（詳細は次のドキュメントを参照してください。鼻や耳などの薄い部分。上記のスクリーンショットのように、バックライトを当てると、これらの領域に赤い光が見えます。 FaceWorksでは、このプロセスを「ディープスキャッタ」と呼びます.FaceWorksは、シャドウマップを使用して光がオブジェクトを通過する距離を見積もることで深い散乱を実現します。その距離に基づいてフォールオフ関数が適用されます。あなたの既存のシャドウマップで動作できるので、追加のレンダリングパスは必要ありません。通常、シャドウマップから厚みを推定するヘルパー関数を提供しています。オルソまたはパースペクティブ投影の標準深度マップ。しかし、VSMやESMなどの他のシャドウフォーマットを使用している場合は、厚み見積もり用の独自のコードを記述する必要があります。シルエットエッジを避けるために、シャドウサンプリング位置に対して内向きの標準オフセットを適用することをおすすめしますPoissonディスクフィルタなどのワイドフィルタを適用して、結果の厚さ値を軟化させます。厚さを指定すると、FaceWorksは厚さとN・Lに基づいてフォールオフ関数を適用します。結果には、通常の拡散照明用語のように、拡散色と明るい色を乗算する必要があります。また、深い散乱結果にテクスチャを乗算して散乱色を制御し、皮膚表面の静脈や骨のような細部を表現することもできます。###将来の機能FaceWorksの将来のバージョンで計画されているその他のレンダリング機能は次のとおりです。人間の皮膚のためのカスタマイズされた鏡面モデル。 - 反射や屈折効果を含む現実的なアイ照明をサポートする機能。 - カスタマイズ可能な拡散プロファイル。現在、人間の皮膚の拡散プロファイルはFaceWorksにハードコードされています。ただし、これをカスタマイズ可能にして、クリーチャースキン、布、紙、ワックスなどの他の素材をシミュレートできるようにしたいと考えています.- OpenGL、モバイル、コンソールを含む追加のグラフィックスAPIとプラットフォームのサポート。 Guide ----------------- FaceWorksをゲームエンジンに統合するには、以下の3つの主なタスクを実行する必要があります。 - アートパイプラインでは、機能概要に記載されているさまざまな事前計算データ。 FaceWorks C APIとDLLには、このデータを構築するためのルーチンが含まれています.-あなたのゲームエンジンはFaceWorksランタイムAPIを使用して、ピクセルシェーダーによって使用される定数バッファーデータを計算する必要があります。これは、FaceWorks DLLのC APIによっても行われます。このデータを定数バッファの1つに含め、シェーダにLUTを公開する必要があります.- FaceWorksを使用するピクセルシェーダでは、FaceWorks Hを＃インクルードする必要があります
ビジネス向け Google 翻訳:翻訳者ツールキットウェブサイト翻訳ツール
-----------------
SL APIは、シェーダにコンパイルされたコードの一種です。設定されたすべてのデータにアクセスし、ライティング方程式を評価するルーチンが含まれています。次のセクションでは、これらの各タスクについて詳しく説明します。###初期化FaceWorks関数を使用する前に、 `GFSDK_FaceWorks_Init（）` 。これにより、ライブラリの内部状態が設定され、ヘッダーとDLLのバージョン番号が一致することが確認されます。### Precomputed Data FaceWorksで使用される事前計算データは、次の2つのカテゴリに分類されます。 - レンダリングされるメッシュのジオメトリに関するデータFaceWorks-事前計算された散乱結果を格納する2つのグローバルルックアップテクスチャ####メッシュデータSSS計算に入力されるFaceWorksは、頂点ごとの曲率およびメッシュごとのUVスケールデータを使用します。 `GFSDK_FaceWorks_CalculateMeshCurvature（）`関数を使用して、曲率値。この関数は、メッシュの頂点の位置と法線、浮動小数点の配列、メッシュの三角形のインデックスを取ります。各頂点の周りの曲率を推定し、ノイズを減らすためにオプションで1つ以上のスムージングパスを適用します（サンプルアプリケーションで2つのパスを使用します）。結果として得られる曲率は、頂点ごとに1つの浮動小数点数の出力配列に格納されます。曲率を頂点バッファに保存し、FaceWorksが使用されるピクセルシェーダで補間された曲率を利用できるようにする必要があります（ただし、メッシュがアニメーションまたは変形すると曲率の値は変更されるはずですが、FaceWorks現在のところ曲率値をリアルタイムでアニメーション化するためのサポートは提供していませんが、実際には曲率を変更する効果に気付くのが難しいと思われますので、メッシュのバインドポーズの曲率を計算することをおすすめします）曲率に加えて、FaceWorksはメッシュごとの平均UVスケールを使用して法線マップをサンプリングするためのミップレベルを較正します。 `GFSDK_FaceWorks_CalculateMeshUVScale（）`関数を使ってメッシュ毎に1つのフロートを平均UVスケールとして計算することができます。このデータをどこかのメッシュの横に保存し、後で設定構造体（the-runtime-apiを参照）を介してFaceWorksランタイムAPIに伝える必要があります.Curvatureには逆の長さの単位があり、UVスケールには長さの単位があります。したがって、実行時にメッシュが拡大縮小されている場合は、UVスケールにメッシュに使用されているのと同じスケールファクタを乗算し、その係数ですべての曲率値を除算する必要があります.NB：頂点位置FaceWorksとのすべてのやりとりを通して同じユニットが一貫して適用されている限り、ルックアップテクスチャファースワークスはまた、2つのルックアップテクスチャに依存します.1つは曲率に関連し、2つのルックアップテクスチャは1つのルックアップテクスチャに依存します。 1つを影にする。これらのテクスチャはメッシュに特有のものではありません。 FaceWorksでレンダリングされたすべてのメッシュで再利用できるように、通常は一度生成してエンジンにインポートすることができます.FaceWorksリポジトリの `samples / lut_generator`ディレクトリには、これらを生成するためのユーティリティがありますLUTを読み込み、.bmpファイルとして保存します。また、 `GFSDK_FaceWorks_GenerateCurvatureLUT（）`と `GFSDK_FaceWorks_GenerateShadowLUT（）`関数を使ってプログラムでLUTを生成することもできます。これらの関数のそれぞれは、生成プロセスのパラメータを詳述するコンフィグレーション構造体を受け取ります。 `GFSDK_FaceWorks_CurvatureLUTConfig`のメンバは以下の通りです： - ` m_diffusionRadius`世界単位での望ましい拡散半径。これは、拡散プロファイルの最も広いガウス分布のシグマであり、人間の肌では2.7 mmです.- `m_texWidth`、` m_texHeight`はピクセル単位の望ましいテクスチャ次元です.- `m_curvatureRadiusMin`、` m_curvatureRadiusMax`テクスチャに表現される最大曲率半径。これらの値は、モデルで見つかった典型的な曲率の範囲を含む範囲に設定します。人間の顔の場合、0.1cm〜10cmが良い選択です。最小半径はゼロにすることはできません（これは透視投影の近い平面のような無限の曲率に対応するためです）。 `GFSDK_FaceWorks_ShadowLUTConfig`のメンバは非常に似ています： - ` m_diffusionRadius`は世界の所望の拡散半径`m_texWidth`、` m_texHeight`はピクセル単位の所望のテクスチャ次元です。 - `m_shadowWidthMin`、` m_shadowWidthMax`テクスチャに表示されるシャドウフィルタの最小/最大の幅です。フィルタ幅とは、シャドウエッジを横切るときにシャドウフィルタが0から1になる距離（ワールド単位）を意味します。最小幅は0にすることはできません。 - `m_shadowSharpening`は、出力シャドウをシャープにする比率です。言い換えれば、これを10に設定すると、実際よりも10倍小さいシャドウフィルタをシミュレートし、フィルタ幅の90％をSSSをシミュレートするために使用できます。RGBA8形式のテクスチャがすべて左から右の上から下のピクセル
ビジネス向け Google 翻訳:翻訳者ツールキットウェブサイト翻訳ツールグ
-----------------
注文。呼び出し元は、出力を保持するのに十分なメモリを割り当てる必要があります。必要なサイズは `GFSDK_FaceWorks_CalculateCurvatureLUTSizeBytes（）`と `GFSDK_FaceWorks_CalculateShadowLUTSizeBytes（）`を呼び出すことで計算できます。アプリケーションがテクスチャの範囲外の曲率や影の幅を使用しようとすると、テクスチャ;あまりにも遠すぎない限り、結果は妥当なままです。サンプルアプリケーションでは、非圧縮RGBA8で保存された512x512のLUTを使用しました。必要に応じて、より小さい解像度も使用できます。圧縮が必要な場合は、DX6でBC6、BC7、またはYCoCgを試してみることをお勧めします。曲率LUTは線形RGBカラースペースで生成され、シャドーLUTはsRGBカラースペースで生成されることに注意することが重要です。DXT1は、スムーズなグラデーションで大量のバンディングを作成するため、推奨されません。 FaceWorksは、テクスチャフォーマットがこれを反映することを期待しています。つまり曲率LUTは `UNORM '形式で、シャドウLUTは` UNORM_SRGB`形式です。 FaceWorksランタイムAPIはこれをチェックし、シェーダーリソースビューが予期された形式でない場合に警告を出します。###ランタイムAPI FaceWorksを使用する描画呼び出しを行う前に、必要なグラフィックス状態を設定する必要があります。 FaceWorksピクセルシェーダAPI。これは、材料ごとの定数バッファデータと、SSSに使用される2つのLUTで構成されています。Materialデータの場合、FaceWorksは定数バッファの1つに組み込むことができるHLSL構造体 `GFSDK_FaceWorks_CBData`を提供します。この構造体は固定サイズで、SSSとディープスキャッタの両方のデータを含んでいます。コンフィギュレーション構造体を記入し、対応する `WriteCBData`関数を呼び出すことで生成することができます（SSSとディープスキャッタ用に1つずつ）。また、2つのLUTをピクセルシェーダと適切なサンプラで使用できるようにする必要があります; #### CB Data SSSに関連するSSSConstantバッファデータは、 `GFSDK_FaceWorks_SSSConfig`構造体に記入し、次に` GFSDK_FaceWorks_WriteCBDataForSSS（） `を呼び出すことで生成できます。 `GFSDK_FaceWorks_SSSConfig`のメンバは以下の通りです： - ` m_diffusionRadius`世界単位での拡散半径を指定します。 - `m_diffusionRadiusLUT`は、LUTの構築に使用される拡散半径です。このパラメータを前の値から分離すると、LUTを再構築することなく、実行時に拡散半径を変更することができます。 FaceWorksは自動的にライティング方程式に必要な調整を行います。 - `m_curvatureRadiusMinLUT`、` m_curvatureRadiusMaxLUT`曲率LUTの作成に使用される最小/最大曲率値（ `GFSDK_FaceWorks_CurvatureLUTConfig`で設定された値と一致する必要があります）.-` m_shadowWidthMinLUT`、 `m_shadowWidthMaxLUT` `シャドウLUTを構築するのに使用される最小/最大シャドー幅値（` GFSDK_FaceWorks_ShadowLUTConfig`で設定された値と一致する必要があります）.- `m_shadowFilterWidth`光源に直接面する表面上のシャドーフィルタのワールド空間幅。つまり、シャドーエッジを横切るときにシャドウ値が0から1になる距離.- m_normalMapSize法線マップのピクセル解像度（フィーチャで説明したように、ぼかしされた標準サンプルのミップレベルを計算するために使用されます） - オーバービュー）。幅と高さが異なる場合は、 `GFSDK_FaceWorks_CalculateMeshUVScale（）`で計算されたメッシュの2つの `m_averageUVScale`平均UVスケールの平均値を使用してください。####深い散乱に関連する深いScatterConstantバッファデータのCBデータ`GFSDK_FaceWorks_DeepScatterConfig`構造体に記入し、次に` GFSDK_FaceWorks_WriteCBDataForDeepScatter（） `を呼び出すことで設定してください。 `GFSDK_FaceWorks_DeepScatterConfig`のメンバは以下の通りです。 - ` m_radius`は、ワールド単位での希望の深い分散半径です。換言すれば、距離光はモデルを透過することができる。これは、ガウスシグマとして使用されます。原則として、この値はSSSに使用される拡散半径と等しくなければならないが、実際にはこれを5〜6mmなどのより大きな半径に設定する方が審美的に好ましい場合が多い - 'm_shadowProjType`シャドーマップこれと次の2つのメンバーは、シャドウマップから厚みを推定するためにFaceWorksヘルパー関数を使用している場合にのみ必要です。自分自身で厚さを見積もっている場合は、 `GFSDK_FaceWorks_NoProjection`に設定します。 - ` m_shadowProjMatrix`シャドーマップ投影行列。行優先順序で格納され、行ベクトル演算規則を前提とします。 - `m_shadowFilterRadius`シャドーマップのUV空間での距離として表現された、厚さの推定値のための望ましいフィルター半径### FaceWorksを使用しているPixel ShaderPixelシェーダーでは、多くの責任がありますが、すべての機能を利用するためには、コードとFaceWorksコードの間を行き来するかなり複雑なコントロールがあります。シェーダは次のことを行う必要があります。基本的な拡散照明： - inter

-----------------
ノーマルマップを通常どおり2回、FaceWorkで計算したミップレベルが高い方に1回サンプルします。FaceWorksに電話して、2つの法線を使って直接拡散照明の項を評価し、曲率。シャドー： - ワイドフィルターを使用してシャドウを評価する.-広いシャドウ結果を使用してシャドウSSS項を評価するためにFaceWorksに電話する。周囲照明用： - 周囲光を評価するために使用する3つの法線を生成するためにFaceWorksに電話する。 FaceWorksヘルパー関数または独自のコードを使用して、シャドーマップから厚さを推定します。 - シャドウマップから厚さを推定します。 FaceWorksを呼び出して、厚さを使って深い散乱光の項を評価する.-すべての項を加え、必要に応じて拡散した色と明るい色を加え、鏡面を追加する。以下のセクションでは、これらのステップについて詳しく説明する。 CurvatureFirst、法線マップをサンプリングするミップレベルを決定するには、 `GFSDK_FaceWorks_CalculateMipLevelForBlurredNormal（）`を呼び出すことができます。この関数は以下を受け入れます： - 定数バッファに渡された `GFSDK_FaceWorks_CBData`構造体 - ミップレベルを計算するテクスチャ、サンプラ、およびUV（この関数はテクスチャサンプリング自体を行わず、D3D11の` CalculateLevelOfDetail （） `メソッドを呼び出します。法線マップを通常どおり1回、結果のミップレベルで1回、2回サンプリングします。直接法線を計算するには、 `GFSDK_FaceWorks_EvaluateSSSDirectLight（）`を呼び出してください。この関数は以下を受け入れます： - 定数バッファに渡された `GFSDK_FaceWorks_CBData`構造体 - 幾何学的法線（補間された頂点法線） - 法線マップからの陰影法線 - 高次のサンプルを持つ法線マップからの、影付き点から光源に向かう単位ベクトル - 頂点から補間された曲率値 - 曲率LUTItのテクスチャとサンプラーオブジェクトは、ベクトルがどの空間にあるかは関係ありません（ビュー、ワールド、ローカル、タンジェントなど）。 ）、それらがすべて同じ空間にある限りです。その結果は、普段と同じように拡散色、明るい色、および影の項で乗算する準備ができているRGB拡散照明項です。換言すれば、これは、標準の拡散照明の飽和（ドット（N、L））係数を置き換えます。この関数は、必要に応じて、異なる光源の結果を計算するために複数回呼び出すことができます。機能の概要、シャドウフィルタリングの詳細はあなた次第です。 FaceWorksはあなたのために実際のフィルタリングを行いません。シャドウフィルタの値が広い場合は、 `GFSDK_FaceWorks_EvaluateSSSShadow（）`を呼び出します。 - GFSDK_FaceWorks_CBData構造体が定数バッファに渡される - 幾何学的法線（補間された頂点法線） - 陰影をつけた点から光源に向かう単位ベクトル - 影の値（0 =完全に陰影、1 =完全に点灯） - シャドーLUTのテクスチャおよびサンプラーオブジェクト結果は、シャドーインバージョンの入力シャドウと、SSSによるシャドーエッジ内の赤色グローを含むRGBシャドーカラーです。これは `GFSDK_FaceWorks_EvaluateSSSDirectLight（）`の結果に拡散色と明るい色を掛け合わせることができます。さらに、入力影の鮮明なバージョンを鏡面やその他の照明用語に使用して、それらが一致するようにしたいでしょうSSS拡散項。 GFSDK_FaceWorks_SharpenShadow（）を呼び出すと、これを簡単に計算できます。 （シャドウLUTを生成するときに使用したのと同じ値、ルックアップテクスチャを参照してください）#### Ambient Light周囲光に使用される3つの法線ベクトルを生成するには、 `GFSDK_FaceWorks_CalculateNormalsForAmbientLight（） `。この関数は、以前に見たシェーディングノーマルとブラーノーマルを受け取り、周囲の光を評価する3つの法線を（出力パラメータを介して）返します。次に、これらの3つの法線から来る周囲光を評価し、 `GFSDK_FaceWorks_EvaluateSSSAmbientLight（）`結果はRGBアンビエントライティングの結果になり、拡散色で乗算する準備ができています。####シャドウマップの厚さを評価するためのFaceWorksヘルパー関数は次のとおりです： - `GFSDK_FaceWorks_EstimateThicknessFromParallelShadowPoisson8（）` `` GFSDK_FaceWorks_EstimateThicknessFromPerspectiveShadowPoisson8（） ` `GFSDK_FaceWorks_EstimateThicknessFromParallelShadowPoisson32（）` `` GFSDK_FaceWorks_EstimateThicknessFromPerspectiveShadowPoisson32（） `は、8個または32個のPoissonディスクタップを使用して、平行投影または透視投影の標準的なデプスマップのケースを処理します。これらの機能を使用するためには、シャドウマップ情報が `GFSDK_FaceWorks_DeepScat
-----------------
--------------------

--------------------
-----------------ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
NVIDIA FaceWorks 1.0
====================

Table of Contents
-----------------

* [Overview](#overview)
* [GitHub Release Notes](#github-release-notes)
* [Contents of the Repository](#contents-of-the-repository)
* [Feature Overview](#feature-overview)
* [Integration Guide](#integration-guide)
* [Version History](#version-history)

Overview
--------

FaceWorks is a middleware library for high-quality skin and eye rendering, inspired by the techniques used in NVIDIA's ["Digital Ira" demo](http://www.nvidia.com/coolstuff/demos#!/lifelike-human-face-rendering).

Digital Ira is a tech demo, created by NVIDIA in collaboration with Dr. Paul Debevec's team at USC, in which a high-resolution head scan and performance capture are rendered as realistically as possible in real-time, using every rendering technique the demo team could think of. FaceWorks, in contrast, aims to be a reusable, production-ready library, with the goal of enabling game developers to match the rendering quality of Digital Ira.

FaceWorks currently provides two main features:

-   A high-quality, efficient subsurface scattering solution, which supports both direct and ambient light.
-   An implementation of deep scattering (translucency) for direct light, based on estimating thickness from a shadow map.

Currently, Direct3D 11 is the only API supported by FaceWorks.

GitHub Release Notes
--------------------

This repository uses DXUT as a git submodule. In order to correctly build the projects, you need to open your git shell in the main repository diretory and initialize the submodules by typing:

`git submodule update --init`

Contents of the Repository
--------------------------

-   `doc/` Documentation images and slides from talks about FaceWorks.
-   `include/` C/C++ and HLSL header files to include into your project.
-   `samples/` source for sample apps demonstrating how to use FaceWorks.
    -   `samples/build/` Project/solution files for VS 2012, 2013 and 2015, with 32-bit and 64-bit builds in each.
    -   `samples/externals/` third-party code used by the sample apps.
    -   `samples/d3d11/` interactive D3D11 sample.
    -   `samples/lut_generator/` command-line utility for building the lookup textures (LUTs) used by the subsurface scattering algorithm.
    -   `samples/media/` models and textures used by the interactive sample.
-   `src/` C/C++ and HLSL source files, as well as VS 2012, 2013 and 2015 project files to build the library, with 32-bit and 64-bit builds in each.

Feature Overview
----------------

This section gives a high-level overview of how FaceWorks' rendering operates and what data it uses. See the integration-guide for instructions for integrating FaceWorks into your game engine.

Alternatively, for a high-level overview of FaceWorks, you can also read the slides from our GTC 2014 talk, available in [pdf](doc/slides/FaceWorks-Overview-GTC14.pdf) and [pptx](doc/slides/FaceWorks-Overview-GTC14.pptx) format.

### Subsurface Scattering

![Comparison of rendering a human face with and without subsurface scattering ([Larger version](doc/images/sss_comparison_large.png))](doc/images/sss_comparison.png)

Subsurface scattering (SSS) is a physical process in which light penetrates into the skin, bounces around and exits at a different point; it's a critical part of the appearance of realistic human skin, since without it, skin looks unnaturally hard and dry, as seen in the screenshots above.

FaceWorks includes an implementation of SSS based on Eric Penner's 2010 SIGGRAPH paper, [Pre-Integrated Skin Shading](http://advances.realtimerendering.com/s2011/Penner%20-%20Pre-Integrated%20Skin%20Rendering%20%28Siggraph%202011%20Advances%20in%20Real-Time%20Rendering%20Course%29.pptx). This has the advantage of requiring only a single rendering pass, which makes it both more efficient than multi-pass methods like texture-space diffusion, and also easier to integrate into a game engine.

In addition, FaceWorks supports SSS for ambient light as well as direct light. Ambient light can be implemented using spherical harmonics, image-based lighting, or any other method; the only requirement is that you be able to evaluate it for different normal vectors in the pixel shader.

Our implementation of SSS is broken down into four components:

**Curvature:** we precompute per-vertex curvature values for each mesh that will have FaceWorks rendering applied. SSS is more apparent in regions of the mesh with greater curvature. In the pixel shader, the curvature and N·L at each point are used to sample a lookup texture (LUT) storing precomputed scattering results.

**Normal maps<!--- -->:** we sample the normal map twice, once as usual and once with a higher mip level, to get a blurred normal corresponding to the SSS radius. The lighting from both normals is combined with the curvature term to arrive at the total diffuse lighting. In order to calculate the correct mip level, FaceWorks also needs to know the UV scale, which we precompute per mesh.

**Shadow edges:** light diffuses from lit areas into shadows, producing a red glow inside the shadow edge. The trick to this is to use a wide shadow filter, then sharpen up the shadows and use the remaining value range to add that red glow. We leave the details of shadow filtering up to you; you can use any method as long as it gives a consistent filter radius (as seen from the light source). Contact-hardening methods are not desirable here.

Given that wide shadow filter value, FaceWorks uses it to sample another LUT storing precomputed scattering results, and this gives the final shadow color.

In principle, the shadow filter width (i.e. the distance over which the shadow value goes from 0 to 1 when crossing a shadow edge, on a surface directly facing the light source) should be at least as wide as the SSS diffusion filter. Note that the radius set in various FaceWorks configuration structs is the sigma of the widest Gaussian in the diffusion profile, so the total filter width is about 6 times larger. For human skin, the diffusion radius is 2.7 mm, so the shadow filter width should ideally be at least 16.2 mm.

However, such a large shadow filter may be impractical for both performance and aesthetic reasons. If you use a narrower filter, the light won’t bleed as far across shadow edges as it physically should, but the SSS result calculated by FaceWorks will still have a smooth falloff and look visually convincing.

**Ambient light:** as mentioned earlier, you can use any type of ambient lighting as long as you can evaluate it for multiple normal vectors. FaceWorks uses the two normal vectors seen previously as well as a third, intermediate normal; ambient light is evaluated for each of these normals, then the three lighting values are combined back together into one.

FaceWorks will evaluate the lighting due to each of these components; you can then multiply the result by the diffuse color and light color, just like the usual diffuse lighting term, and add a specular model of your choice.

### Deep Scatter

![Comparison of rendering a human face with and without deep scatter ([Larger version](doc/images/deep_scatter_comparison_large.png))](doc/images/deep_scatter_comparison.png)

While SSS models the diffusion of light locally along the surface of the skin, light can also scatter through the bulk of a model in thin areas such as the nose and ears. This produces the red glow seen in these areas when backlit, as in the screenshots above. In FaceWorks we call this process "deep scatter".

FaceWorks implements deep scatter by estimating the distance light would have to travel through an object, using the shadow map; then a falloff function is applied based on that distance. It can work with your existing shadow maps, so no additional rendering passes are needed.

We provide helper functions to estimate thickness from the shadow map in common cases. Standard depth maps with either ortho or perspective projections. However, if you're using some other shadow format, such as VSM or ESM, then you'll have to write your own code for thickness estimation.

We recommend applying an inward normal offset to the shadow sampling position, to help avoid silhouette edge issues, and also applying a wide filter, such as a Poisson disk filter, to soften the resulting thickness values.

Given the thickness, FaceWorks applies a falloff function based on the thickness and N·L. The result should be multiplied by the diffuse color and light color, just like the usual diffuse lighting term. You can also multiply the deep scatter result by a texture to control the scattering color, and represent details like veins and bones under the skin surface.

### Future Features

Additional rendering features planned for future versions of FaceWorks include:

-   An extension to deep scatter to support ambient light as well as direct.
-   A customized specular model for human skin.
-   Features to support realistic eye lighting, including reflection and refraction effects.
-   Customizable diffusion profiles. Currently, the human skin diffusion profile is hard-coded into FaceWorks; however, we'd like to make this customizable, so that other materials such as creature skin, cloth, paper, wax, etc. can be simulated.
-   Support for additional graphics APIs and platforms, including OpenGL, mobile, and consoles.

Integration Guide
-----------------

To integrate FaceWorks into your game engine, there are three main tasks you'll have to accomplish:

-   In your art pipeline, you'll need to generate the various precomputed data mentioned in the feature-overview. The FaceWorks C API and DLL contain the routines for building this data.
-   Your game engine will need to use the FaceWorks runtime API to calculate the constant buffer data used by the pixel shaders. This is also done by a C API in the FaceWorks DLL. You'll need to include this data in one of your constant buffers, and also expose the LUTs to the shader.
-   In pixel shaders where you want to use FaceWorks, you'll need to `#include` the FaceWorks HLSL API, which is a blob of code that gets compiled into your shaders. It contains the routines to access all the data that's been set up and evaluate the lighting equations.

The following sections discuss each of these tasks in detail.

### Initialization

Before using any FaceWorks functions, you'll need to call `GFSDK_FaceWorks_Init()`. This sets up the internal state of the library and sanity-checks that the version numbers of the header and DLL match.

### Precomputed Data

The precomputed data used by FaceWorks falls into two categories:

-   Data about the geometry of meshes that will be rendered with FaceWorks
-   Two global lookup textures storing precomputed scattering results

#### Mesh Data

As input to its SSS calculations, FaceWorks uses per-vertex curvature and per-mesh UV scale data.

The `GFSDK_FaceWorks_CalculateMeshCurvature()` function can be used to calculate the curvature values. This function takes the mesh vertex positions and normals, as arrays of floats, as well as the mesh's triangle indices. It estimates the curvature around each vertex, then optionally applies one or more smoothing passes to reduce noise (we use two passes in the sample app). The resulting curvatures are then stored to an output array, one float per vertex. You'll need to store the curvatures in a vertex buffer, and make the interpolated curvature available in pixel shaders where FaceWorks is to be used.

(Note that in principle, curvature values should change if a mesh animates or deforms; however, in FaceWorks we don't currently provide any support to animate curvature values in real-time. In practice, we suspect it's difficult to notice the effect of changing curvature in common cases, so we recommend simply calculating curvature for the bind pose of a mesh.)

In addition to curvature, FaceWorks uses a per-mesh average UV scale to calibrate the mip level for sampling the normal map. The `GFSDK_FaceWorks_CalculateMeshUVScale()` function can be used to calculate the average UV scale, one float per mesh. You should store this data alongside the mesh somewhere, then later communicate it to the FaceWorks runtime API via a configuration struct (see the-runtime-api).

Curvature has units of inverse length, and UV scale has units of length; therefore, if a mesh is scaled at runtime, you should multiply the UV scale by the same scale factor used for the mesh, and divide all the curvature values by that factor.

NB: it doesn't matter what units are used for vertex positions, as long as the same units are applied consistently throughout all your interactions with FaceWorks. The length values used for computing curvature, building the LUTs, in the runtime configuration structs, and in the pixel shader should all be expressed in the same units.

#### Lookup Textures

FaceWorks also depends on two lookup textures, one related to curvature and one to shadows. These textures are not specific to a mesh; you can most likely generate them once and import them into your engine as regular RGB textures, to be re-used across all meshes rendered with FaceWorks.

In the `samples/lut_generator` directory in the FaceWorks repository, there is a utility for generating these LUTs and saving them out as .bmp files. You can also generate the LUTs programmatically using the `GFSDK_FaceWorks_GenerateCurvatureLUT()` and `GFSDK_FaceWorks_GenerateShadowLUT()` functions. Each of these functions accepts a configuration struct detailing the parameters to the generation process.

The members of `GFSDK_FaceWorks_CurvatureLUTConfig` are as follows:

-   `m_diffusionRadius` the desired diffusion radius, in world units. NB: this is the sigma of the widest Gaussian in the diffusion profile, which is 2.7 mm for human skin.
-   `m_texWidth`, `m_texHeight` the desired texture dimensions in pixels.
-   `m_curvatureRadiusMin`, `m_curvatureRadiusMax` the desired min/max radius of curvature to be represented in the texture. Set these to a range that encompasses the typical range of curvatures found in your models. For human faces, 0.1 cm to 10 cm is a good choice. Note that the minimum radius cannot be zero (as this would correspond to infinite curvature it's a bit like the near plane of a perspective projection).

The members of `GFSDK_FaceWorks_ShadowLUTConfig` are very similar:

-   `m_diffusionRadius` the desired diffusion radius, in world units.
-   `m_texWidth`, `m_texHeight` the desired texture dimensions in pixels.
-   `m_shadowWidthMin`, `m_shadowWidthMax` the desired min/max shadow filter width to be represented in the texture. By filter width, we mean the distance (in world units) over which the shadow filter will go from 0 to 1 when crossing a shadow edge. The minimum width cannot be zero.
-   `m_shadowSharpening` the ratio by which the output shadow is sharpened. In other words, if this is set to 10, we'll simulate a shadow filter 10x less wide than the real one, and the other 90% of the filter width is available to simulate SSS.

Both textures are generated in RGBA8 format, in left-to-right top-to-bottom pixel order. The caller is responsible for allocating sufficient memory to hold the output; the required size can be calculated by calling `GFSDK_FaceWorks_CalculateCurvatureLUTSizeBytes()` and `GFSDK_FaceWorks_CalculateShadowLUTSizeBytes()`.

Note that if the application tries to use curvatures or shadow widths outside of the ranges represented in the textures, we will simply clamp to the edge of the textures; the results will remain plausible as long as you're not too far outside the range.

In the sample app, we used 512x512 LUTs, stored in uncompressed RGBA8. Smaller resolutions can also be used if desired. If compression is necessary, we suggest trying BC6, BC7, or YCoCg in DXT5; DXT1 isn't recommended, since it will create a great deal of banding in the smooth gradients.

It's important to note that the curvature LUT is generated in linear RGB color space, while the shadow LUT is generated in sRGB color space. FaceWorks expects their texture formats to reflect this, i.e. the curvature LUT should be in a `UNORM` format and the shadow LUT in a `UNORM_SRGB` format. The FaceWorks runtime API will check for this and issue warnings if the shader resource views are not in the expected formats.

### The Runtime API

Before doing a draw call that uses FaceWorks, you'll need to set up the graphics state needed by the FaceWorks pixel shader API. This consists of per-material constant buffer data, as well as the two LUTs used for SSS.

For material data, FaceWorks provides an HLSL struct, `GFSDK_FaceWorks_CBData`, which you can incorporate in one of your constant buffers. This struct has a fixed size and contains the data for both SSS and deep scatter. It can be generated by filling out a configuration struct and calling the corresponding `WriteCBData` function (one each for SSS and deep scatter).

You'll also need to make the two LUTs available to the pixel shader, as well as an appropriate sampler; the LUTs should be sampled with bilinear filtering and clamp addressing.

#### CB Data For SSS

Constant buffer data related to SSS can be generated by filling out a `GFSDK_FaceWorks_SSSConfig` struct, then calling `GFSDK_FaceWorks_WriteCBDataForSSS()`. The members of `GFSDK_FaceWorks_SSSConfig` are as follows:

-   `m_diffusionRadius` the desired diffusion radius, in world units.
-   `m_diffusionRadiusLUT` the diffusion radius used to build the LUTs. Separating this parameter from the previous one allows the diffusion radius to be varied at runtime, without rebuilding the LUTs. FaceWorks will automatically make the necessary adjustments in the lighting equations.
-   `m_curvatureRadiusMinLUT`, `m_curvatureRadiusMaxLUT` min/max curvature values used to build the curvature LUT (should match the values set in `GFSDK_FaceWorks_CurvatureLUTConfig`).
-   `m_shadowWidthMinLUT`, `m_shadowWidthMaxLUT` min/max shadow width values used to build the shadow LUT (should match the values set in `GFSDK_FaceWorks_ShadowLUTConfig`).
-   `m_shadowFilterWidth` world-space width of the shadow filter, on a surface directly facing the light source. In other words, the distance over which the shadow value goes from 0 to 1 when crossing a shadow edge.
-   `m_normalMapSize` pixel resolution of the normal map (used to calculate the mip level for the blurred normal sample, as mentioned in the feature-overview). If its width and height differ, use the average of the two.
-   `m_averageUVScale` average UV scale of the mesh, as computed by `GFSDK_FaceWorks_CalculateMeshUVScale()`.

#### CB Data For Deep Scatter

Constant buffer data related to deep scatter can be set by filling out a `GFSDK_FaceWorks_DeepScatterConfig` struct, then calling `GFSDK_FaceWorks_WriteCBDataForDeepScatter()`. The members of `GFSDK_FaceWorks_DeepScatterConfig` are as follows:

-   `m_radius` the desired deep scatter radius, in world units. In other words, the distance light can penetrate through the model. This is used as a Gaussian sigma. In principle, this value should equal the diffusion radius used for SSS, but in practice it's often more aesthetically pleasing to set this to a larger radius, such as 5--6 mm.
-   `m_shadowProjType` the type of projection matrix used for the shadow map. This, and the next two members, are only needed if you're using the FaceWorks helper functions for estimating thickness from the shadow map. Set to `GFSDK_FaceWorks_NoProjection` if you're doing the thickness estimate yourself.
-   `m_shadowProjMatrix` the shadow map projection matrix, stored in row-major order and assuming a row-vector math convention; transpose your matrix if you're using different conventions.
-   `m_shadowFilterRadius` the desired filter radius for the thickness estimate, expressed as a distance in shadow map UV space.

### In The Pixel Shader

Pixel shaders making use of FaceWorks have many responsibilities, and there's a fairly intricate chain of control leading back and forth between your code and FaceWorks code in order to make use of all the features. The shader needs to do the following.

For basic diffuse lighting:

-   The interpolated per-vertex curvature needs to be available in the pixel shader.
-   Sample the normal map twice, once as usual and once with a higher mip level calculated by FaceWorks.
-   Call FaceWorks to evaluate the direct diffuse lighting term using the two normals and the curvature.

For shadows:

-   Evaluate shadows using a wide filter.
-   Call FaceWorks to evaluate the shadow SSS term using the wide shadow result.

For ambient lighting:

-   Call FaceWorks to generate the three normals to use to evaluate ambient light.
-   Evaluate ambient light at those three normals.
-   Call FaceWorks to combine those three lighting values to get the ambient diffuse lighting term.

For deep scatter:

-   Estimate the thickness from the shadow map, using either a FaceWorks helper function, or your own code.
-   Call FaceWorks to evaluate the deep scatter lighting term using the thickness.
-   Add up all the terms, factor in the diffuse color and light color where needed, and add specular.

The following sections discuss these steps in detail.

#### Normals and Curvature

First, to determine the mip level at which to sample the normal map, you can call `GFSDK_FaceWorks_CalculateMipLevelForBlurredNormal()`. This function accepts:

-   The `GFSDK_FaceWorks_CBData` struct passed down in a constant buffer
-   The texture, sampler, and UV at which to calculate the mip level

(Note that this function doesn't do any texture sampling itself, just calls D3D11's `CalculateLevelOfDetail()` method.)

Sample the normal map twice, once as usual and once with the resulting mip level. Unpack both normals, convert them from tangent to world space, etc. just as you usually would.

To calculate direct diffuse lighting, call `GFSDK_FaceWorks_EvaluateSSSDirectLight()`. This function accepts:

-   The `GFSDK_FaceWorks_CBData` struct passed down in a constant buffer
-   The geometric normal (i.e. interpolated vertex normal)
-   The shading normal from the normal map
-   The blurred normal, from the normal map with the higher-mip sample
-   The unit vector from the shaded point toward the light source
-   The curvature value interpolated from the vertices
-   The texture and sampler objects for the curvature LUT

It doesn't matter what space the vectors are in (view, world, local, tangent, etc.), as long as they're all in the same space.

The result is the RGB diffuse lighting term, ready to be multiplied by the diffuse color, light color, and shadow term just as you would normally do. In other words, it replaces the `saturate(dot(N, L))` factor in standard diffuse lighting.

This function can be called several times to calculate results for different light sources, if necessary.

#### Shadows

As mentioned in the feature-overview, the details of shadow filtering are up to you; FaceWorks does not do the actual filtering for you.

Once you have the wide shadow filter value, call `GFSDK_FaceWorks_EvaluateSSSShadow()`. This function accepts:

-   The `GFSDK_FaceWorks_CBData` struct passed down in a constant buffer
-   The geometric normal (i.e. interpolated vertex normal)
-   The unit vector from the shaded point toward the light source
-   The shadow value (0 = fully in shadow, 1 = fully lit)
-   The texture and sampler objects for the shadow LUT

The result is the RGB shadow color, including both the sharpened version of the input shadow as well as the red glow inside the shadow edge due to SSS. This can be multiplied by the result from `GFSDK_FaceWorks_EvaluateSSSDirectLight()`, the diffuse color and the light color.

Additionally, you will likely want to use the sharpened version of the input shadow for specular and other lighting terms, so that they'll match the SSS diffuse term. You can call `GFSDK_FaceWorks_SharpenShadow()` to quickly calculate this; it accepts the input shadow value as well as the sharpening factor (the same value used when generating the shadow LUT; see lookup-textures).

#### Ambient Light

To generate the three normal vectors used for ambient light, call `GFSDK_FaceWorks_CalculateNormalsForAmbientLight()`. This function accepts the shading normal and blurred normal seen previously, and returns (via output parameters) the three normals at which to evaluate ambient light.

You can then evaluate the ambient light coming from each of those three normals, and pass the RGB results to `GFSDK_FaceWorks_EvaluateSSSAmbientLight()`. The result will be the RGB ambient lighting result, ready to be multiplied by the diffuse color.

#### Deep Scatter

The FaceWorks helper functions for evaluating the shadow map thickness are as follows:

-   `GFSDK_FaceWorks_EstimateThicknessFromParallelShadowPoisson8()`
-   `GFSDK_FaceWorks_EstimateThicknessFromPerspectiveShadowPoisson8()`
-   `GFSDK_FaceWorks_EstimateThicknessFromParallelShadowPoisson32()`
-   `GFSDK_FaceWorks_EstimateThicknessFromPerspectiveShadowPoisson32()`

These handle the cases of a standard depth map with either parallel or perspective projection, using either 8 or 32 Poisson disk taps. In order to use these functions, shadow map information must have been provided to FaceWorks in the `GFSDK_FaceWorks_DeepScatterConfig` struct (see cb-data-deep-scatter). Each of these functions accepts:

-   The `GFSDK_FaceWorks_CBData` struct passed down in a constant buffer
-   The depth texture and sampler to use
-   The UVZ coordinates in \[0, 1] at which to sample

The functions all return the thickness in world units.

Note that these functions do not include the inward normal offset to the shadow sampling position that we recommended in the feature-overview. You'll need to apply this offset to the position yourself. The sample app included with FaceWorks demonstrates this.

Whether you use these helper functions or your own thickness estimate, you can pass the result to `GFSDK_FaceWorks_EvaluateDeepScatterDirectLight()`. This function accepts:

-   The `GFSDK_FaceWorks_CBData` struct passed down in a constant buffer
-   The blurred normal seen previously
-   The unit vector from the shaded point toward the light source
-   The thickness value, in world units

It returns the scalar intensity of deep scatter, which you can multiply by the desired deep scatter color (e.g. a deep red for human skin, possibly modulated by a texture to represent under-skin details like veins and bones), the diffuse color, and the light color.

### Error Handling

Most FaceWorks functions return a `GFSDK_FaceWorks_Result` code, which is either `GFSDK_FaceWorks_OK` or an error code that coarsely indicates what went wrong.

For more detail, FaceWorks returns verbose error messages using the `GFSDK_FaceWorks_ErrorBlob` struct, which encapsulates a message string. If you create one of these structs and pass it in to FaceWorks functions that accept it, we will allocate the memory and append errors and warnings to it as necessary. You can then display or log the message string as desired.

When you're done with the error blob, call `GFSDK_FaceWorks_FreeErrorBlob()` to free the memory. You can also provide allocator callbacks via the `m_allocator` member of the error blob, in which case FaceWorks will use that allocator for the message string memory.

Version History
---------------

###FaceWorks 1.0 (March 2016) - First opensource release

-   Codebase migrated to latest version DXUT (as submodules from GitHub), and upgraded to use latest DX11 and DirectXMath.
-   Fixed a problem when UV components when triangle coords were degenerate
-   Code cleanup and doxygen comments
-   Converted documentation to markdown

###FaceWorks 0.8 (April 2014)

-   Deep scattering for direct light.
-   Helper functions for estimating thickness from standard types of shadow maps, for deep scattering.
-   Subsurface scattering for ambient light.
-   Various improvements to sample app.

###FaceWorks 0.5 (January 2014)

-   Preintegrated subsurface scattering supported in D3D11, for skin rendering.
-   APIs to generate mesh data (curvature, UV scale) used by preintegrated SSS.
-   APIs to generate lookup textures (LUTs) used by preintegrated SSS.
