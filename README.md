# MikuMikuDayo  

![0](https://github.com/pennennennennennenem/MikuMikuDayo/assets/56704844/4900961c-f1a2-4fe2-978d-4e57b4b2a8b0)

## 概要
なんかこれじゃないPMXファイル用のレンダラです  
sdPBRなどのシェーダを作るにあたって新しいレンダリング手法を試す時には、お手本が無いと結果が正しいのかどうかわかりませんので、そのお手本を作るためのモノです  
現状では動画を作って遊ぶための物にはなり得ませんから、DanceではなくてDayoです 
一応VMDファイルによるモーションを読み込んでポーズを付ける事は出来ます(VPDには対応していません)


## つくりかけです
そういうことでよろしく  


## 起動方法
エクスプローラ上で`x64\release\MikuMikuDayo.exe`をダブルクリック！  

- Windows11環境で作っているのでWindows10でどうなるかはわかりません
- DLLが無くて起動しない場合は[こちら](https://learn.microsoft.com/ja-jp/cpp/windows/latest-supported-vc-redist?view=msvc-170)
- どれをDLしたらいいのか分からない場合は多分[これ](https://aka.ms/vs/17/release/vc_redist.x64.exe)です


## 操作方法
マウスやテンキーでの操作については画面に表示される通りなのでメニューについて  

### Fileメニュー
`File`メニューでスカイボックス,モデル,モーションの読み込みが出来ます。モデルはPMX2.0形式のみ対応しています(PMDやXは不可)

### `View`メニュー
- `denoise`でIntel OIDNを使用したデノイズのON/OFFが出来ます  
- `spectral rendering`で分光を考慮したレンダリングをON/OFFできます  

また、以下の3種類からシェーダを選択できます  
- `preview shader` 材質の違いやカメラの設定をほとんど考慮しない軽いシェーダです
- `pathtracing shader` 片方向パストレースによる割合軽量ですが十分な表現力の有るシェーダです
- `bidirectional pathtracing shader` 双方向パストレースによる重いけれどコースティクスなど複雑な表現のできるシェーダです


## VMDファイル読み込み時のコードページについて

VMDファイルに格納されているボーン名・モーフ名を示す文字列はANSIでエンコードされているため、VMDファイル内にボーン名として格納されているバイト列と表示される文字列との対応は環境により異なる可能性があります。これに対応できるよう、VMDファイルが作成された際に使われていたコードページが何であるかを`config.json`内の`vmd_codepage`で指定できるようになっています  

デフォルトでは日本語(Shift-JIS)を示すコードページ932が指定されていますが、もし他国語版モデル専用のモーションが正しく扱えない場合は、`config.json`内の`vmd_codepage`を変更すると解消できるかもしれません  

現状ではいずれにしてもPMXファイルでのボーン名・モーフ名と完全一致した場合のみ正しく認識する実装になっているので、ANSIエンコードされた状態で15バイトを超えるボーン名・モーフ名を指すキーフレームは扱えませんのでご注意ください  


## 材質の設定方法

PMXファイルに含まれる物体の材質名に応じて金属や肌などの質感を切り替えています。ですから、モデルをコピーしたうえでPMXeditorで材質名の変更を適宜行う必要があるでしょう  

`config.json`をいじると材質名に含まれるキーワードと材質の対応付けを変えられますが、エラーチェックは雑なので正しく読み込めるjsonファイルでないとどんなエラーが出るか分かりませんから、バックアップを取ってからいじって下さい  

デフォルトの`config.json`では以下のような設定になっていますので、PMXファイルをいじる場合の参考にしてください
- 材質名に`肌`が含まれる場合、表面下散乱を考慮した、赤色がよく伸びる材質になります
- 材質名に`金`が含まれる場合、よく研磨された金属的な材質になります
- 材質名に`ガラス`または`氷`が含まれる場合、透過・屈折を考慮した材質になります。また、現実の材料以上に派手に分光を起こします
- 材質名に`髪`が含まれる場合、凹凸感があり、異方性反射によるハイライトの付いた材質になります
- 材質名に`白目`が含まれる場合、弱い発光要素を持ちます  
  アニメ調モデルでは白目は特殊な構造をしている場合があり、正直に計算するとそれが原因で白目の部分が黒くなってしまう事が多々ありますので、それを補填するための処置です
- 材質名に`灯`が含まれる場合、発光要素が大きく加算され、光源として使えるようになります
- 材質名に`ビーム`が含まれる場合、指向性の強い強力な光源とみなされます  
  法線方向に狭い角度に大きな光度を持ちます
  `bidirectional pathtracing shader`以外ではあまり良く扱えません


jsonオブジェクトは以下の構成になっています
- `default_material` 特に指定のない場合のマテリアルパラメータについての指定
- `material` マテリアルプリセットの定義
- `rule` PMXモデルの材質名に含まれるキーワードに対して、マテリアルプリセットのどれを使うかという対応付け

### default_material

例 "default_material": { "IOR":[1.5,0], "roughness":[0.5,0.5] },  

使用できるマテリアルパラメータ名とパラメータの記述法は以下の通りです  
大文字・小文字は区別されます  
正しく認識できないマテリアルパラメータ名を指定してもエラーは出ずに認識されないだけになるため、正確に書いて下さい

- "roughness":[数値T,数値B]  
  表面の粗さ(マイクロファセットの向きの不均一さ)です  
  2つの数値はそれぞれ接線,従法線方向に対応し、0～1で指定します  
  数値T=数値Bの時、鏡面反射が等方性になり、数値T≠数値Bの時、鏡面反射が異方性になります
- "IOR":[数値A,数値B]  
  材質の屈折率です  
  実際の屈折率はA+(B/λ[nm])^2としてコーシーの分散公式によって計算され、波長依存性を表現できます  
- "autoNormal":数値  
  テクスチャの明度勾配に応じた法線マップを自動生成します  
  数値が大きいほど凹凸感が出ます  
  おおむね0～1程度を想定していますが、1より大きくしても問題はありません
- "category":"metal" or "subsurface" or "glass"  
  マテリアルのカテゴリを金属・表面下散乱・ガラスから選択します  
  指定が無い場合は標準マテリアルになります
- "emission":[数値r,数値g,数値b]  
  マテリアルの発光要素をr,g,bの順に配列で指定します
- "cat":[数値r,数値g,数値b,数値scale]  
  カテゴリ固有パラメータの配列です  
  現在は`subsurface`のみ対応しており、以下のような意味になります  
  [媒質内での光の平均自由行程r, g, b, 平均自由行程のスケール]  
  平均自由行程のスケールが1.0の時、平均自由行程パラメータ1.0は10cmに対応し、平均自由行程のスケールが0.01の時、平均自由行程パラメータ1.0は1mmに対応します
- "light":数値  
  材質の反射率の数値倍の値が発光要素に加算されます
- "lightFalloff":数値  
  発光要素は発光面の法線方向に対してスポットライト状に広がりを持ちます  
  広がりの範囲をラジアン単位で指定します  
  省略された場合、π/2になります  
  あまり狭い場合は双方向パストレーサでないとうまく扱えません
- "lightHotspot":数値  
  発光要素が減衰せずに広がる範囲です  
  ラジアン単位で0～π/2の範囲で指定します  
  lightFalloff以下の値にしてください  
  省略された場合、π/2扱いになります

### material

"name"の後にマテリアル名を文字列で指定する必要があります。あとはdefault_materialと同様のオブジェクトを配列で指定します  

### rule
例 [ { "keyword":["金","メタル","metal"], "material":"metal" }, ... ]  

"keyword"配列に指定されたキーワードのどれか一つを材質名に持つ場合、"material"に指定されたマテリアルが適用されます  

つくりかけですし、この辺りは後々整理されて次のバージョンでは全く使えなくなるでしょうから今日の所は、てきとうに遊んでください  

## 構築方法
`src`ディレクトリにソースコードが入っています  
レンダラ自体のソースはdayo.cppの1ファイルだけです  
全く整理されてないので無理して自分でビルドしなくてもいいです  

シェーダのソースコードは`MikuMikuDayo\hlsl`に入っています  
VisualStudio2022でのビルドを念頭にしています  
Windowsデスクトップアプリケーションで空のプロジェクトを作成してdayo.cpp, YRZ.ixx, PMXLoader.ixxを追加し、プロジェクトのプロパティ → C/C++ → 言語 で`C++言語標準`をC++20、`準拠モード`を「はい」にして下さい  

ビルドするために以下も必要です  

- [DirectXTex](https://github.com/microsoft/DirectXTex)(desktop_win10を選択)
- [DirectX12Tk](https://github.com/microsoft/DirectXTK12)収録のSimpleMath.cpp,SimpleMath.h,d3dx12.h
- [BulletPhysics](https://github.com/microsoft/DirectXTex)
- [Intel open image denoise](https://www.openimagedenoise.org/)(OIDN)

DirectX TexはNuGetでインストールするのが簡単でしょう  

SimpleMath.cpp, SimleMath.h, d3dx12.hについてはDirectX12Tkから3つのファイルをコピーしてプロジェクトのソースファイルに入れればOKです  

BulletPhysicsは付属のcmakeのオプションだとDirectXTex等と一緒に使えないライブラリが出力されるのでPMXLoader.ixxの先頭付近に書いてある方法を使ってビルドしています。詳しいニキはcmakeのオプションいじって解決するとコンパイルがちょっと早くなっていいかもね  

OIDNについては上記リンクよりダウンロードしたoidn-2.2.2.x64.windows.zipを展開し、oidn.hppにインクルードパスを通し、OpenImageDenoise.libにライブラリパスを通せばOKです  


## 使用範囲

MITライセンスで公開されたソフトウェアです

このソフトウェアを使用して作った画像についてはこのソフトウェアの作者は何ら権利も義務も持ちませんから、各自の責任で使って下さい  

## 謝辞

`sample`フォルダに添付されているskybox用HDRI `lebombo_2k.hdr` は、[polyhaven.com](https://polyhaven.com/hdris)にて公開されている物を利用させていただきました  

PMXLoaderの物理演算対応にあたってはbenikabocha氏[saba](https://github.com/benikabocha/saba)を参考にさせていただきました

誠にありがとうございます  


## FAQ

- ひこうきしか有りませんが、ダヨーさんはどこですか  
ひこうきには[すず式ミクダヨー](https://www.nicovideo.jp/watch/sm18358421)さんが丁度よく載せられるようになっていますから、PMXEditor等を使って載せてあげたらいいと思います  
ひこうきは僕が作ったモデルで、[CC0](https://creativecommons.jp/sciencecommons/aboutcc0/)で公開しております  

- 動かないんですけど  
DXR対応ビデオカードが無いと動かせないんですけど  

- DXR対応ビデオカードのはずなんですけど  
そんなら動くはずなんですけど、ヨソでテストはしてないんですけど  
exeファイルと同じディレクトリにyrz.logというファイルが出来ているのでメモ帳で読んだらなんかヒントになる事が書いてあるかもしれません  

- MMD関連ソフトですか？  
それは間違いないと考えています  

- MMD互換ソフトですか？  
ミクさんとミクダヨーさんの関係に似ていないこともないと思います  

- MMDですか？  
MikuMikuDayoです  
略す時はMMDayoと書いて下さい  
Dayoはそれ以上略せないんダヨー  
