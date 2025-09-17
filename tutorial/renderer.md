# 第四章 レンダラの作り方
MikuMikuDayoエフェクトの最高峰？レンダラの作り方について説明します。

レンダラは最低限やらないといけない事というものが多いのでミニマルな実装でも大変です。長丁場になるかもしれませんが、コツコツといきましよう。

このチュートリアルはヨソでは読めないMikuMikuDayo用エフェクト作成法についての文書ですから、初級3DCGプログラミング講座的な内容は全部**わかっているもの**としてすっとばしていますので、分かんなかったらAIに聞くとかしてね。

このチュートリアルでは**DXRの組み込み関数**という物がしばしば出てきます。これはDirectX Raytracingで用意されている関数であり、名前空間なしで、レイトレーシングシェーダの特定の関数内から使う事が来ます。ぼくが作った**よろずDXR**が提供しているわけではないのでDXRとよろずDXRを混同しないようにしてください。

## レンダラの仕事
レンダラはモデルデータやカメラなどのパラメータから画面を描画する**レンダリング**が仕事のほぼ全てですが、他にも以下の仕事があります

- デノイザのためのデータの出力
- Gバッファの出力

これらはレンダリングさせる部分がちゃんとできていればオマケで出せるような物でもあります。

とりあえずは、画面を作る所から始めていきましょう、それ以外の部分を組んでなくても一部のポストプロセスやデノイザが正しく動作しなくなるだけなので、レンダラだけが動作している状態では特に影響がありません。

## 下準備
まずは、MikuMikuDayoを起動して、`postprocess/ToneMapper/ACESTonemap.pmx`を読み込みましょう。レンダリング結果が正しくても色がおかしいのでは浮かばれません。

では、サンプルのコードを眺めてみましょう。デノイザとGバッファ出力に対応していないシンプルな状態のコードは`sample/RendererTuorial.fxdayo`として保存されています。実行するにはこのファイルを`renderer/`フォルダにコピーしてからMikuMikuDayoを起動すると、`Renderer`メニューから選択・起動できるようになります。

## 画面をつくる

画面を作るにはカメラの真似をするのが簡単ですから、かんたんなカメラのシミュレータみたいな物をこしらえる事にします。

暗箱にフィルムが置いてあり、フィルムの反対側の壁に小さな、計算上は大きさ0とみなせるサイズのピンホールが空いている、ピンホールカメラを考えましょう。このピンホールの位置が`Dayo::CameraPosition`としてDayo側から渡ってる値の物理的な意味であるとします。

<img src="g/camera.png">

フィルムのサイズは縦が24ミリ、幅は画面の縦横比(アスペクト比といいます)に従って24*アスペクト比で求めます。そして、フィルムからピンホールまでの距離は画角によって決ます。フィルムとピンホールが近ければ広角になり、フィルムとピンホールが遠ければズームした状態になります。カメラの箱のサイズは自由自在に変えられるわけです。すごいね！そして、画角についての係数は`Dayo::CameraFoV`として得られます。これは`hlsl/cb.hlsli`に書いてあるように射影行列から得られている値です。

以上より、輝度を求めたい画面上の1点から、ワールド座標でのフィルム上の位置を求めるにはこの、サンプルコードの上の方で定義されている`FilmPos`ヘルパー関数を使います。

```c
//画面上の点uv(左上隅が(0,0),右下隅が(1,1))に対応するフィルム上の点のワールド座標を計算する
float3 FilmPos(float2 uv)
{
    const float H = 24.0 * 0.5 / 80.0;    //1MMD長さ=80mm
    float3 fp;  //ビュー座標系でのフィルム上の点の位置
    fp.xy = (uv*2-1) * float2(-H * Dayo::AspectRatio, H);
    fp.z = -H * Dayo::CameraFoV;
    return mul((float3x3)Dayo::ViewMatrix,fp) + Dayo::CameraPosition;
}
```

なんか料理番組のノリですが、使いまわせる関数でもありますから、新しいレンダラを作りたくなったらその都度コピペして必要な所は書き換えて使ってください。なにしろMITライセンスですからね。

画面上の位置を`uv`として、このFilmPos関数によってピンホールの位置に飛んでいく光線(レイ)の向き`rd`は簡単に求められます。

```
float3 rd = normalize(Dayo::CameraPosition - FilmPos(uv));
```

「どうせnormalizeするんだからフィルムの高さの半分を1として計算してもいいんじゃね？」と思うかもしれませんけど、ピンホールカメラじゃなくて開口の大きさによるピンボケの効果(DoF)を入れるとか今後の拡張を考えるとフィルムのサイズを実寸で指定するのは悪くない選択肢です。

後は、このレイを追っていけば画像の完成です。

光源からじゃなくてカメラから光線が発射されるというのは奇妙に感じるかもしれませんが、光源からレイを発射してピンホールを通ってたまたまフィルムに当たった光をカウントするような作り方にしてしまうと、カメラの箱にピンホールじゃなくてよほどデカい穴をあけてもなかなかフィルムに光がぶっかってくれません。いつまでたっても黒いままの画面をただ眺め続けるだけになってしまいます。光の反射についての振る舞いというのは光の発射元→発射先に辿った場合でも発射先→発射元に逆に辿る場合でも同じように扱える**相反性**という素晴らしい性質が成り立っています。それを利用してカメラからレイを発射(?)して光源にぶっかったら明るい画素なのだ、として取り扱えば良いじゃない？ということにしています。ぼくだけじゃなくてみんな大体そうです。相反性の綻びについての議論というのもあるんですけど、むつかしい話なので今回は置いときます。

<img src="g/recipro.png">

レイは物体の表面に当たるとあっちこっちに反射しますから、何百回・何千回とレンダリングをして、その平均をとることで真の画像に近づいていくようにします。**モンテカルロパストレーシング**という方法です。とはいえ本当に何百回もレンダリングしないといけないのは大変ですから、16回くらいレンダリングしたらそこにデノイザ(ノイズ除去フィルタ)を掛けて、なんとか見れる絵として仕上げるというアプローチで行くわけです。

「昔のレイトレーシングのCGってそんな何千回もレンダリングしなくて良くなかったっけ？」という記憶のある人もいるでしょう。そうなんです。そのかわり、昔のCGは何でもツルツルテカテカしてたのを覚えているでしょうか？1回のレンダリングで絵が求まる代わりに光はあっちこっちに反射するという大切な事実を意図的に伏せて、反射は鏡と同じようなツルテカ反射しか扱わなかったんです。拡散反射なんかは本当にいい加減に近似されていました。当時のみんなのおうちのマイコンではとても扱いきれなかったからです。でも今は、1枚のCGを昔のツルテカレイトレで描くくらいだったらリアルタイムでもできちゃったりするようになっちまいました。ですから、何百回か描かせて平均取ってでもレイがあっちこっちに反射する様子まできちっとシミュレーションしよう！と、そんな時代です。なんてこった！


このチュートリアルではレンダリングに必須の知識である測光量とか放射量といった光を扱うための単位についての話を省略しています。こうした話はちゃんとした実装、ちゃんとした理解には欠かせない基礎ですが、とりあえず手を動かして物を作ってみない事にはピンと来ない知識でもあります。そういうわけで、このチュートリアルでは光を測るための単位の話はあまり厳密にやりません。チュートリアルを一通り終えて、しっかりした知識が欲しくなってからしかるべきソースを当たるのが良いと思います。

## json部の書き方

それでは、実際のコードの書き方の説明に入ります。まずはjson部から。

```json
#include <resources.hlsli>
#include <yrz.hlsli>

[YRZFX]
{
    "fx": {
        "cereal_class_version": 0,
        "category": "render",
        "samplers" : [{"name":"samp"}],
        "passes" : [
            {
                "name":"Main", "type":"raytracing",
				"raygenShader": "RayGeneration",
                "missShader": ["Miss"],
                "hitGroup": [ { "closestHit": "ClosestHit", "anyHit": "AnyHit" }],
                "maxPayloadSize": 128, "maxAttributeSize": 8, "maxRecursionDepth": 1
            }
        ]
    }
}
[HLSL]
```
HLSL部のインクルード文からjson部の終わりまでを見ていきましょう。レンダラで必須のインクルードファイルは`<resources.hlsli>`で、`<yrz.hlsli>`は任意ですが、今回はこっちもかなりいい仕事をしてくれます。

`fx`オブジェクトのパラメータを見ていきましょう。
- **category**  
今回はレンダラを作るので、`render`を指定します。
- **passes**  
1個のレイトレーシングパスが定義されています。レイトレーシングパスを作るためのシェーダは4つもあるのと、他のパラメータもあるので複雑そうに見えます。この中を丁寧に読んでいきましょう

```json
"name":"Main", "type":"raytracing",
"raygenShader": "RayGeneration",
"missShader": ["Miss"],
"hitGroup": [ { "closestHit": "ClosestHit", "anyHit": "AnyHit" }],
"maxPayloadSize": 128, "maxAttributeSize": 8, "maxRecursionDepth": 1
```

- **name** いつものパス名です
- **type** レイトレーシングパスなので`raytracing`を指定します。

そして、以下の3つ(実質4つ)のパラメータでレイトレーシングにまつわるシェーダを定義します

- **raygenShader**  
レイを発射するためのraygeneration shaderのエントリポイント名です。まずはこれが呼ばれます。
- **missShader**
発射されたレイが何にも当たらなかった時に呼ばれるmiss shaderのエントリポイント名です。
- **hitGroup**
発射したレイがポリゴンに当たった時に起動されるシェーダ群とエントリポイントの組みを配列で指定しています。
    - **closestHit**  
    レイとヒットしたポリゴンのうち、発射元から最寄りの物が確定した時に呼ばれるclosestHitシェーダのエントリポイント名です。
    - **anyHit**  
    レイとポリゴンがヒットした時にポリゴンのα値などから「本当にヒットした扱いにしていい？スルーしとく？」などの判断をするためのanyHitシェーダのエントリポイント名です。

missShaderとhitGroupはraygenShaderの作り方によっては複数必要な事もあるので配列になっているんですが、今回は1種類ずつしか使いません。

さらに、以下の3つのパラメータでレイトレーシングパイプラインの設定をします

- **maxPayloadSize**  
後で説明するPayload構造体のサイズです。なるべく小さくしとくと高速化されていいんだそうですけど、開発中は設計が定まらずに増えたり減ったりするので余裕を持たせても良いでしょう。
- **maxAttributeSize**  
anyHitShader、closestHitShaderに渡される。ヒットした物体上の位置を格納するための構造体のサイズです。今回は三角ポリゴン群を扱うシェーダしか考えていないので8にしてください。
- **maxRecursionDepth**  
各レイトレーシングシェーダからは更にTraceRay()という関数でレイトレーシングシェーダを起動できます。今回はそのような入れ子になった呼び方はしないので1でいいです。

レイトレーシングシェーダはこのチュートリアルで使う物だけでも4つも有って大変そうですが、4つの機能の分かれ方は非常に明快でそれぞれのシェーダ自体はそんなに長くないですから、安心してください。

レイトレーシングシェーダについてもうちょっと詳しく見たいという人は[こちらのMicrosoftの公式ドキュメント](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html)をどうぞ。

## HLSL部のコード

まず、HLSL部で使う構造体を定義しておきましょう。
```c
//ペイロード。最後にどこにレイが当たったかの情報
struct Payload {
    uint imo;    //ヒットした物体番号(Dayo::Inai何にもヒットしてない)
    uint iface;  //面番号
    float2 st;  //重心座標(uvだとテクスチャ座標と間違うのでstと書いてある)
    float t;    //最後にレイを撃った位置からヒットした場所までの距離
    uint xi;    //サイコロ振り用
};

//PBRマテリアル
struct Material {
    float alpha;    //不透過度
    float3 albedo;  //反射率
};

//レイがぶっかった表面についての情報
struct Patch {
    float3 pos;     //ワールド座標での位置
    float3x3 TBN;   //接空間(tangent,binomal,normalの順)
    Material m;     //マテリアル
};
```

大体コメントにある通りなんですが、`Payload`構造体はレイトレーシングに欠かせません。これがレイに乗っかってぶん投げられて、上手い事レイトレーシングをする事で知りたかった情報を載せて帰って来る、そういう仕組みになっています。

ヘルパー関数などはヒマな時にでも読んどいてくれればいいです。とりあえず今は**ああ、そういう便利な関数あるのね**くらいでOK

### raygenerationシェーダ

では一番下のraygenerationシェーダから見ていきましょう。
```c
[shader("raygeneration")]
void RayGeneration()
{
	Payload payload = (Payload)0;

     //1.初期化
    const int nRays = 8;    //レイの発射回数
	Payload payload = (Payload)0;
    uint2 pix = DispatchRaysIndex().xy;
    float3 beta = 1;    //パスのスループット…このパスを辿った光はどれだけの倍率でカメラまで届くのか？という割合
    float3 Lt = 0;      //パスの寄与…パスを辿って光源→カメラまで実際に届いた光の輝度
    bool directSky = false;

    //2.レイ生成
    float3 ro = Dayo::CameraPosition;
    float3 rd = normalize(ro - FilmPos((pix+RNG.xy) / DispatchRaysDimensions().xy));

    //3.レイをトレース
    for (int i=0; i<MaxBounce; i++) {
        RayDesc ray;
        ray.Origin = ro;
        ray.Direction = rd;
        ray.TMin = 1e-3;    //数値誤差対策に少し浮かせて発射するのがミソ
        ray.TMax = 1e+5;    //10万MMDは本家の遠方クリッピング面までの距離と同じ
        TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 0,0,0, ray, payload);

        if (payload.imo == Dayo::Inai) {
            //何も当たらなかったので空を仰ぐ
            Lt += FetchSkybox(rd) * beta;
            directSky = (i==0);
            break;
        } else {
            //当たったので反射先を追う
            Patch P = GetPatch(payload.imo, payload.iface, payload.st);
            float pdf;
            ro = P.pos;
            rd = YRZ::SampleCosWeightedHemisphere(P.TBN, RNG.xy, pdf);
            float3 brdf = P.m.albedo / PI;
            float geom = abs(dot(P.TBN[2],rd));
            beta *= geom * brdf / pdf;
        }
    }

    //4.結果の格納
    float4 cur = float4(Lt,directSky ? 0 : 1);
    Dayo::RTOutput[pix] = lerp(Dayo::RTOutput[pix], cur, 1.0/(iSample+1));
}
```
なんか長いんですけど、4つのパートに分かれていて各パートはそんなでもないでしょう？

```c
[shader("raygeneration")]
void RayGeneration()
{
```
まずはお約束の部分ですが、レイトレーシングのシェーダというのはエントリポイント名の上に[shader("なんのシェーダなのか")]という形式で断り書きをする必要があります。そうしないとパスの定義でちゃんと書いてあってもRayGeneration()という関数がraygeneration shaderのための関数であるとコンパイラに分かってもらえません。

```c
    //1.初期化
    const int nRays = 8;
	Payload payload = (Payload)0;
    uint2 pix = DispatchRaysIndex().xy;
    float3 beta = 1;    //パスのスループット…このパスを辿った光はどれだけの倍率でカメラまで届くのか？という割合
    float3 Lt = 0;      //パスの寄与…パスを辿って光源→カメラまで実際に届いた光の輝度
    bool directSky = false;
```
次は初期化というかスタッフ紹介的な部分です。

- **nRays**  
レイが物体の表面に当たって反射する様子を追いかけるにあたって最大で何回レイを発射するのかを定義します。今回はとりあえず8としていますが、この値を変更すると見た目と重さのバランスを取ることが出来ます。
- **payload**  
さっきまで話題に出ていたPayload構造体の変数です。これがレイトレーシングの旅に出て荷物を運んで帰ってきます。その荷物が何で、どう使うのかはすぐ明らかになります。
- **pix**  
DispatchRaysIndex()というDXRの組み込み関数を使って今起動されたレイトレーシングシェーダがどのピクセルについてのレイなのかという事を受け取ります。(0,0)-(画面の横画素数-1,画面の縦画素数-1)の範囲の値になります。この場合の画面、というのはMikuMikuDayoの`Renderer`ウィンドウの表示領域の事です。
- **beta**  
レイが物体の表面に当たって反射すると、物体の反射率分だけレイは「弱まる」事になります。例えばカメラから出てすぐに明るさ1の光源に当たったとしたら「明るさ1！」として報告できますが、反射率0.5の物体に反射してから明るさ1の光源に当たったら、「明るさ0.5！」として報告されるべきです。そのような倍率を`beta`という変数に記録します。これはあくまでざっくりした説明ですから、もう少し詳しい事はコードを見ながら説明します。
- **Lt**  
レイがカメラまで運んできた光の輝度を格納します。
- **directSky**  
MikuMikuDayoの背景かどうかを判定するためのフラグです。カメラから出たレイが何にも当たらなかったら**背景に当たりました**としてフラグを立てます。

次は、短いけどレイ生成です。

```c
    //2.レイ生成
    float3 ro = Dayo::CameraPosition;
    float3 rd = normalize(ro - FilmPos((pix+RNG.xy) / DispatchRaysDimensions().xy));
```

2行しかないようでヘルパー関数`FilmPos()`を使っているので実質もうちょい長いんですが、`FilmPos()`はレイが運んできた輝度によって色を付けたい画面上の位置から、フィルム上の1点に相当する位置のワールド座標を返す関数です。`DispatchRaysDimensions()`はDXRの組み込み関数で画面の画素数が入っています。今までのチュートリアルでも出てきた`Dayo::Resolution`でも一緒です。

`RNG`というのは0以上1未満のランダムな値を4つ、float4型で返してくれるヘルパーマクロです。詳しい実装はサンプルコードに書いてありますから、時間の有る時にでも読んでいただくとして、`RNG`と書くたびに違う乱数が返ってくるので大変重宝します。ちなみに、`RNG`とはRandom Number Generatorの略です。

この`RNG`をpixに足すことで、フィルム上の位置を探す時にサンプル毎にランダムに0～1ピクセル分ズレた位置を探すことになります。つまり、アンチエイリアシングがこれだけで出来るのです。

フィルム上の点からピンホールの位置(`Dayo::CameraPosition`)へ進むベクトルを正規化してやると、レイの進行方向`rd`が求まります。

ピンホールの位置からレイが発射されるようにすると分かりやすいですから、レイの原点を`ro`として保持します。

さて、お次は山場のレイをトレースする部分です

```c
    //3.レイをトレース
    for (int i=0; i<nRays; i++) {
        RayDesc ray;
        ray.Origin = ro;
        ray.Direction = rd;
        ray.TMin = 1e-3;    //数値誤差対策に少し浮かせて発射するのがミソ
        ray.TMax = 1e+5;    //10万MMDは本家の遠方クリッピング面までの距離と同じ
        TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 0,0,0, ray, payload);

        if (payload.imo == Dayo::Inai) {
            //何も当たらなかったので空を仰ぐ
            Lt += FetchSkybox(rd) * beta;
            directSky = (i==0);
            break;
        } else {
            //当たったので反射先を追う
            Patch P = GetPatch(payload.imo, payload.iface, payload.st);
            float pdf;
            ro = P.pos;
            rd = YRZ::SampleCosWeightedHemisphere(P.TBN, RNG.xy, pdf);
            float3 brdf = P.m.albedo / PI;
            float geom = abs(dot(P.TBN[2],rd));
            beta *= geom * brdf / pdf;
        }
    }
```

`nRays`回のループを回して1回につき1本レイを発射し、物体の間をレイが跳ね回るようにします。

レイが飛び回った軌跡の事を**パス**(path)と呼びます。エフェクトを作るパス(pass)と似ていますが綴りが違います。

レイを追いかけてパスを作るので**パストレーシング**と呼ばれる方法です。レイトレーシングの応用の一つです。チュートリアルで作るレンダラがいきなり応用です。

ループの中はどうなってるんでしょうか？まずは`TraceRay()`の所まで読みましょう

```c
    RayDesc ray;
    ray.Origin = ro;
    ray.Direction = rd;
    ray.TMin = 1e-3;    //数値誤差対策に少し浮かせて発射するのがミソ
    ray.TMax = 1e+5;    //10万MMDは本家の遠方クリッピング面までの距離と同じ
    TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 0,0,0, ray, payload);
```

まずRayDesc構造体ににレイの原点`ro`とレイの向き`rd`と、レイを追跡する範囲(TMin～TMax)を指定します。

そして、`TraceRay()`を呼んでいるだけです。`TraceRay()`はDXRの組み込み関数で、`payload`はレイトレの結果書き換えられます。

`Dayo::TLAS`というのはTop Level Acceleration Structureの略でレイトレーシングの対象になるポリゴン群についての情報を収めたオブジェクトで、MikuMikuDayoではMMDモデル群から作られた1つのオブジェクトしかないので`TraceRay()`の第一引数は基本的に`Dayo::TLAS`一択となります。

他の引数については[こちら](https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/traceray-function)を参照して下さい。

ここで、Payload構造体をもう一度見てみましょう。

```c
//ペイロード。最後にどこにレイが当たったかの情報
struct Payload {
    uint imo;    //ヒットした物体番号(Dayo::Inaiで何にもヒットしてない)
    uint iface;  //面番号
    float2 st;  //重心座標(uvだとテクスチャ座標と間違うのでstと書いてある)
    float t;    //最後にレイを撃った位置からヒットした場所までの距離
    uint xi;    //サイコロ振り用
};
```
これらの情報が`TraceRay()`の結果「どの物体の、どの面の、どこでレイが止まったのか？」という情報で埋められます。raygeneration以外の3つのシェーダでその仕事をやりますから、後で説明します。

それを踏まえて続きをif文の上半分まで読みましょう

```c
    if (payload.imo == Dayo::Inai) {
        //何も当たらなかったので空を仰ぐ
        Lt += FetchSkybox(rd) * beta;
        directSky = (i==0);
        break;
    } else { ...
 ```

`payload.imo`にレイがぶつかったモデル番号が入りますが、`Dayo::Inai = 0xFFFFFFFF(-1)`が入っていれば「何にも当たらなかった」という事にしています。この場合、レイは背景のどこかに飛んで行ったことになりますから、背景、つまりskyboxという巨大な発光体から光が得られる、という事になりました。

光とカメラがパスでつながったのです。プッピガァン！！とくっついてエネルギーが流れている様子を想像してください。プッピガァン！！させまくる双方向パストレーシングという更に難しいテクニックもありますが、チュートリアルの範囲を大きく逸脱するので今回は取り上げません。気になる人は`BDPT.fxdayo`を読んでください。

さて、テンション上がったところで`Lt`に`FetchSkybox()`ヘルパー関数で得たskyboxの輝度に`beta`で倍率を掛けて足しこみ、ループを終わらせます。ループカウンタ`i`が0の時はカメラから直接背景を見ている状態になっていますから、その時はフラグを立てておきます。後で使います。

さて、いよいよ絵作りのキモの部分です。

```c
...
    } else {
        //当たったので反射先を追う
        Patch P = GetPatch(payload.imo, payload.iface, payload.st);
        float pdf;
        ro = P.pos;
        rd = YRZ::SampleCosWeightedHemisphere(P.TBN, RNG.xy, pdf);
        float3 brdf = P.m.albedo / PI;
        float geom = abs(dot(P.TBN[2],rd));
        beta *= geom * brdf / pdf;
    }
}
```

レイが物体にヒットしたら、`GetPatch`ヘルパー関数を使って当たった点(の周辺のごく小さな面)についての情報を得ます。`Patch`構造体は以下のように接空間や位置、反射率などの情報を格納しています。

接空間って耳慣れないですか?法線はなんとなくわかると思います。面の向いてる方向ですね。この法線に対して互いに垂直な2本のベクトルを接線(tangent)と従法線(binormal)といい、それら三本のベクトルの組みによって張られる空間の事を接空間と呼びます。慣例的に接線が接空間のX軸、従法線がY軸、法線がZ軸になります。なので`TBN`と略しています。タイピングの早い人は`TangentSpace`とか書いたりするみたいですけど。

法線だけでなく接空間、T,B,Nとして3本のベクトルが全部分かっていると色々と計算の幅が広がって便利なんです。すぐ後で使いますよ。

```c
//PBRマテリアル
struct Material {
    float alpha;    //不透過度
    float3 albedo;  //反射率
};

//レイがぶっかった表面についての情報
struct Patch {
    float3 pos;     //ワールド座標での位置
    float3x3 TBN;   //接空間(TBN[0]=tangent,TBN[1]=binormal,TBN[2]=normal)
    Material m;     //マテリアル
};
```

パッチの位置を`ro`に代入して、次にレイが発射される始点にするのはいいんですが、次にレイが発射される方向`rd`の方がなんかヤバい事になっています。早速、接空間も出てきました。

```
    rd = YRZ::SampleCosWeightedHemisphere(P.TBN, RNG.xy, pdf);
```
この長い名前のYRZに入っている関数は何をしているのかというと、レイが物体の表面に当たった後の反射先をサンプリング(乱数に基づいて選択)しています。今回のチュートリアルで作る物体の反射は拡散反射(ランバート反射)だけを想定していて、光が一度物体の中に入って法線方向の半球上のどこかランダムな方向に出ていきます。こうして計算した材質はテカりが全く無く、物体上の同じ点ならばどの方向から見ても一定の輝度に見えます。

法線方向のどこかランダムに、という事であれば半球面上の一方向を等確率で得れば良さそうですが、この関数には**CosWeighted**としてなぜか「コサインで重みが付けられた」という関数名になっています。これは「ランダムではあるけど法線方向にはなるべく頻繁に、接線・従法線方向にはなるべく少なくサンプルが飛ぶように」確率にイカサマをしているという事です。なんでそんなイカサマをするのか、正々堂々とやれ、と思うかもしれませんけど、意外とこれが重要なポイントです。とりあえず、これで得られたサンプルが反射先`rd`であり、`pdf`にサンプルの確率密度が入ります。

確率密度というのは、そのサンプルのレア度の逆数(コモン度？)くらいに思っといてください。イカサマをせずに半球上のどこかに完全にランダムにレイを反射させる場合は、半球は立体角2π[sr]で、その中から1つサンプルが取れるということで、確率密度は`rd`がいくつでも1/2π[sr⁻¹]になります。今回はイカサマをしているので法線をNと書くと、確率密度は(rd・N)/π[sr⁻¹]になります。つまり、`rd`がN方向に近いほど確率密度は高くなって最大1/π[sr⁻¹]になり、接線や従法線方向に近いほど確率密度は低くなって最小で0[sr⁻¹]に限りなく近づいていきます。実際そのような頻度で`rd`が得られるよう、`SampleCosWeightedHemisphere()`関数は頑張っているのです。

以上を踏まえてパスのスループット`beta`を更新します。

```c
    float3 brdf = P.m.albedo / PI;
    float geom = abs(dot(P.TBN[2],rd));
    beta *= geom * brdf / pdf;
```

`beta`というのは基本的には今までレイがぶつかってきた物体の反射率の積、と考えてもらえばいいんですが、物体自体の性質である反射率`brdf`のほかに物体に対する光の差し込み具合によって届く光が弱まる事を表す**幾何項**とさっき言ったイカサマ補正係数である**確率密度**も関わってきます。

`brdf`は反射率を計算した結果を入れる変数で、今回のように拡散反射しか考えない場合は拡散反射率をρとした場合、`rd`に関わらずρ/πで求まります。今回はわざと光の単位とか測り方についての詳しい話を省略していますから、気になる人はぜひ各自で調べてください。

**幾何項**`geom`というのはCGのレンダリングをちょっと齧った人なら親の顔より見た**N・L**の事です。法線と光の差し込んでくる方向の内積分だけ、物体の表面に伸びて薄まる分を勘定するための項です。

<details>
<summary>←鋭い人はsaturateじゃなくてabsになっている事に気づくかもしれませんが</summary>
レイトレではガラスのような透過を取り扱う場合でもなければ基本的に物体の裏側から光が入ってくる(裏側へレイが出ていく)ことはありません。しかし、表側から入っているはずの光が法線のスムージングの影響で裏側から入ってきているとみなされる事はあるのでそれを補填するためにこのようなインチキをしています。これはあくまでインチキであって、物理的な正しさを担保するものではありませんが、特に法線マップなどをやり始めると見た目は顕著に改善されます。
</details>

さて、**確率密度**`pdf`ですが`beta`をさらに割っています。レア度の逆数で割っているのですがら、つまり、betaにサンプルのレア度を掛けている事になります。

<details>
<summary>
←レアイテムは価値が高いと考えると腹落ち感ありますけど、どういうことか？
</summary>

実は確率密度というのはどのくらいの範囲の代表としてサンプルを拾ってきたのか、という大事な値なのです。確率密度が低いレアなサンプルの場合、「広い範囲を探さないとそのサンプルは取れない」という事で広い範囲を代表するサンプルになりますから、そのサンプルは高いウェイトを持ってレンダリング結果に反映されます。逆に、確率密度が高いコモンなサンプルの場合、「狭い範囲を探せばそのサンプルは取れる」という事で狭い範囲しか代表しておらず低いウェイトを持って結果に反映されます。

レンダリングにおいては平均値に近い結果をコンスタントに出せるのが望ましいので、高い寄与を持ってくるサンプルは確率密度を高く(頻繁にサンプルされるかわりウェイトを低く)、低い寄与を持ってくるサンプルには確率密度を低く(めったにサンプルされないかわりにウェイトを高く)設定してやるというのがコツになります。

なんかめんどうだなぁ、一様にサンプリングしてpdfは1/2πで固定で良くね？と思うかもしれませんから、ここでどういうメリットがあるのか詳しく見てみましょう。

先ほど述べた通り、`SampleCosWeightedHemisphere()`で返ってくる`pdf`は(rd・N)/πでした。そのように偏りのある、イカサマをした`rd`の選び方をしているのです。

```
    beta *= geom * brdf / pdf;
```

ですから、右辺のそれぞれの項を展開すると

```
    beta *= rd・N * P.m.albedo / PI / ((rd・N)/PI);
```

つまり…

```
    beta *= P.m.albedo;
```

これと同じことになります。分かりますでしょうか？これがイカサマの効果です。イカサマのおかげで、`beta`は**ランダムに変化するrdの影響を全く受けないという事になりました**

これがもし、一様にrdをサンプリングする方法だったらどうなるでしょうか？pdfは1/2πで固定ですから、

```
    beta *= geom * brdf / pdf;
    →
    beta *= rd・N * P.m.albedo / PI / (0.5/PI);
    →
    beta *= 2 * rd・N * P.m.albedo;
```

このように、`rd`のランダム性に影響を受けて、最悪で`beta`は反射するつど2倍の値になっちまいます。`beta`が増えるってどういう事だよと思うかもしれませんけど、モンテカルロ法というのは「本当は滅茶苦茶広い範囲からサンプルをかき集めて平均したいけど、時間の都合で1つしかサンプルを取れません、なので今は1つで勘弁してください、その代わりに後で集めた奴を平均します」という計算法です。ですから、計算したい範囲が広くなるほど、広い範囲からかき集める代わりに、拾ってきたサンプルに大きなウェイトを掛けて計算しないと辻褄が合わない事になります。

`beta`に反射率`brdf`と幾何項`geom`を掛けるのは直感的ですが、さらに`pdf`で割るのはどういうことかというと、**今このパスがどれだけ広い範囲の代表としてサンプルを拾ってるのか**という事を`beta`に反映しているのです。サンプリングに工夫をしない場合は反射するつど反射先の候補は増えて範囲も広くなりますから`beta`も増えるというのは本来自然な事なのです。そこでイカサマによって反射先の候補を搾り`beta`がクソほど大きくなるのをなんとか抑えるのが賢い方法です。今回の場合は、`SampleCosWeightedHemisphere()`を使って法線方向にある程度絞って探索している効果が得られているのです。

実際、これは見た目にもインパクトを与えますから、本当か？と疑う暇があるなら自分で書き換えてみたらいいよ！半球内を一様にサンプリングするよう変えるには`YRZ::SampleUniformHemisphere`として名前が変わるだけなので、すぐ試せます。

このように確率にイカサマをしてサンプリングを有利に進める方法を**importance sampling**(重点的サンプリング)といいます。
</details>

以上でループ1回分の計算の説明が終わり、更新した`ro`と`rd`を使って更にレイを飛ばします。

こうして計算した結果`Lt`を`Dayo::RTOutput`に格納したら終わりです
```c
    //4.結果の格納
    float4 cur = float4(Lt,directSky ? 0 : 1);
    Dayo::RTOutput[pix] = lerp(Dayo::RTOutput[pix], cur, 1.0/(iSample+1));
```
まず、α値はレイが背景に直撃した時は0をセットし、そうではなく物体に当たってる時は1をセットしています。**レンダラは背景が表示されている箇所はα=0とする**という決まりがあります。あれ？半透明とかは？いいの？と思うかもしんないですけど、何回もレンダリングして平均を取っているので、αは0か1しか扱わず、そこは確率的に中間の値に収束します。これは残りのシェーダも見ると明らかになります。

そして、いよいよ`Dayo::RTOutput`オブジェクトに書き込むとレンダリング結果を格納した、という事になります。

```
lerp(Dayo::RTOutput[pix], cur, 1.0/(iSample+1));
```
というコードがちょっと面白い所でして、ConstantBufferに乗っている`iSample`の値を重みとして今までの`RTOutput`と現フレームの内容`cur`の値をブレンドしています。

`iSample`は出力時や動作モードが**サンプルためろ**になっている時、0から始まってサンプルの蓄積が進むにつれて1ずつ値が増えます。

1. iSampleが0の時、curのウェイトは1でRTOuputを埋める
2. iSampleが1の時、curのウェイトは1/2 RTOuput(iSample=0の時の内容)のウェイトは1/2
3. iSampleが2の時、curのウェイトは1/3 RTOuput(iSample=0,1の時の内容を半々で混ざた物)のウェイトは2/3 → iSample=0の時のウェイトは1/3,iSample=1の時のウェイトは1/3

…

と続いていくことで、何フレームの平均を取るのか言う知識が事前に無くても全フレームの平均値をRTOutputに格納できる事になります。巧妙ですね。

skyboxに使っているHDR画像にsRGBリニア色空間で表現された物理的な輝度に比例した値が記録されていると仮定すると、こうして計算した結果も同じくsRGBリニア色空間で表現された物理的な輝度に比例した値になっています。レンダラの大事な要件の1つとして、**出力される輝度情報はsRGBリニア色空間で表現されている事**が挙げられます。

<details>
<summary>
sRGBリニアについての補足
</summary>
基本的にsRGB色空間というのはガンマ補正された色空間です。ガンマ補正後は物理的な輝度とは非線形な色空間になります。一方、sRGB-linearやlinear-light sRGBと呼ばれるガンマ補正前の色空間も広く使われており、MikuMikuDayoでは後者のガンマ補正前の色空間であるsRGBリニア色空間をレンダラの出力色空間として採用しています。<br>
<b>リニア色空間</b>はいずれもCIE1931に定められるXYZ3刺激値に対する線形変換によって定義されますが、どのような線形変換を使うかによって同じシーンから算出された輝度であっても具体的な数値は変わってくることになります。MikuMikuDayoでは線形変換としてsRGBで定められる行列を使い、その白色点はD65になります。これはBT.709リニアと等価です。<br>
最終的な出力にあたってはガンマ変換と共にスワップチェーンのフォーマットに変換する作業が必要になりますが、これはMikuMikuDayoが内部的に行っています。将来的にHDRディスプレイ用スワップチェーンへの出力などが簡単(エフェクト作成者やユーザ側では設定項目1つとトーンマッパーの変更程度でHDRディスプレイ出力対応可能)になる事を見込んでいます。
</details>

いやあ、長かったなあ、これでおうちに帰れる…ダヨ…

まだくたばってはいけません。あと3つのシェーダの解説もしますヨー。

### anyhitシェーダ
```c
[shader("anyhit")]
//AnyHitShaderは始点から近く→遠くの順に起動される保証はない
void AnyHit(inout Payload payload, Dayo::Attribute attr)
{
	uint imo = InstanceID();
	uint iface = PrimitiveIndex();
    bool frontface = (HitKind() == HIT_KIND_TRIANGLE_FRONT_FACE);

    Patch p = GetPatch(imo, iface, attr.uv, frontface);
    if (p.m.alpha < RNG.x)
        IgnoreHit();
}
```
だいぶん短くてホッとしますね。残りの2つもかんたんです。安心してください。

このシェーダは「レイが何かポリゴンに当たったんだけどどうしたらいい？」という問い合わせに答えるための物で、読み書きの対象になるPayload構造体と、レイがヒットしたポリゴンの重心座標が納められたDayo::Attribute構造体が渡ってきます。

まず、`frontface`という変数にDXRの組み込み関数`HitKind()`を使って三角形の表側からレイが進入したか？の判定を書き入れます。それを`GetPatch`への入力として使います。`GetPatch`の中身はヒマな時にでも見て頂くといいんですけど、片面ポリゴンを弾くための処理を兼ねています。

そして、`GetPatch`で得たポリゴンのα値を元に、`RNG`の出目と比較して透過率と同じ確率でポリゴンをスルーするように組んでいます。これもDXRの組み込み関数ですが`IgnoreHit()`を呼ぶと「このポリゴンはスルーします」と表明したことになります。これを呼ばずに関数を抜けると、「このポリゴンとヒットしました」と表明したことになります。

半透明ポリゴンに対してはスルーするか、ヒットするかという事を確率にゆだねるので、ヒットしたポリゴンについては必ずα=1の不透明としてレンダリングすれば良い事になります。半透明の効果は何回もレンダリングすることで醸し出されるのです。こうするのがモンテカルロパストレーサの実装としては簡単かつ便利です。

### closesthitシェーダ
```c
[shader("closesthit")]
void ClosestHit(inout Payload payload, Dayo::Attribute attr)
{
	payload.imo = InstanceID();
	payload.iface = PrimitiveIndex();
	payload.st = attr.uv;
	payload.t = RayTCurrent();
}
```
レイの発射元から一番近くでヒットしたポリゴンはこれだ！というのが確定したら呼ばれるシェーダです。今回は「これです！」とそのままpayloadに書き込むだけ、raygenerationに仕事は丸投げです。

`InstanceID()`も`PrimitiveIndex()`も`RayTCurrent()`もDXRの組み込み関数です。

組み方によってはここで`TraceRay`を更に撃って再帰的に反射を追いかけるという考え方もありますが、今回はraygenerationが頑張るパターンで作りました。


### missシェーダ
missシェーダは超簡単です

```c
[shader("miss")]
void Miss(inout Payload payload)
{
    payload.imo = Dayo::Inai;
}
```
何にも当たらなかったのだから、`imo`に`Dayo::Inai`をセットして、「当たりませんでした」という事を報告させるだけです

さて、ついに！4つのシェーダの解説が全部終わりました！しかし、まだ話は続きます！！！！！

## デノイザとGBufferのためのデータの出力

次は、デノイザのためのデータを作る所を説明します。今の状態でMikuMikuDayoの`View`メニューから`デノイザを使う`をチェックすると、画面が真っ黒になります。デノイザへの入力が空だから、デノイザも空の出力を返すためです。

めんどいなあと思うかもしれませんが、Intel open image denoiserの威力はかなりの物で、必要なサンプル数が激減して出力のための時間が大幅に短縮されますから、多少ともモンテカルロアプローチを採用しているレンダラならば対応は必須と言っていいでしょう。

幸い、このための拡張工事は大した手間ではありません。必要な情報は今までの過程でほとんど簡単に得られる状態になっているからです。

改造後のコードは`sample/RendererTuorial2.fxdayo`にありますから、それを覗いてみてください。まず、json部の違いから説明します

```json
{
    "fx": {
        "cereal_class_version": 0,
        "category": "render",
        "samplers" : [{"name":"samp"}],
        "passes" : [
            {
                "name":"Main", "type":"raytracing",
				"raygenShader": "RayGeneration",
                "missShader": ["Miss"],
                "hitGroup": [
                    { "closestHit": "ClosestHit", "anyHit": "AnyHit" },
                    { "closestHit": "ClosestHit", "anyHit": "AnyHitGB" }
                ],
                "maxPayloadSize": 128, "maxAttributeSize": 8, "maxRecursionDepth": 1
            }
        ]
    }
}
```

`hitGroup`が2つになりました。`ClosestHit`は同じものを使いまわしていますが、`AnyHit`が`AniHitGB`としてもう一つあります。

```c
[shader("anyhit")]
//AnyHitShaderは始点から近く→遠くの順に起動される保証はない
void AnyHitGB(inout Payload payload, Dayo::Attribute attr)
{
	uint imo = InstanceID();
	uint iface = PrimitiveIndex();
    bool frontface = (HitKind() == HIT_KIND_TRIANGLE_FRONT_FACE);

    Patch p = GetPatch(imo, iface, attr.uv, frontface);
    if (p.m.alpha < Dayo::AlphaThreshold)
        IgnoreHit();
}
```

`AniyHitGB()`は以上のようになっていて、コピペ元の`AnyHit()`との違いはレイがヒットしたポリゴンを見逃す基準が違うだけです。元のコードでは乱数に基づいて見逃すかを決めていましたが、ここでは`Dayo::AlphaThreshold (=0.5)`未満なら見逃すという事になっています。これはGBufferの作り方の要件です。

そして`RayGeneration()`も少し加筆が必要です。

```c
...
[shader("raygeneration")]
void RayGeneration()
{
	Payload payload = (Payload)0;
    uint2 pix = DispatchRaysIndex().xy;

    //レイ生成
    const int nRays = 8;    //レイの発射回数
    float3 ro = Dayo::CameraPosition;
    float3 rd = normalize(ro - FilmPos((pix+RNG.xy) / DispatchRaysDimensions().xy));
    float3 beta = 1;    //パスのスループット…このパスを辿った光はどれだけの倍率でカメラまで届くのか？という割合
    float3 Lt = 0;      //パスの寄与…パスを辿って光源→カメラまで実際に届いた光の輝度
    bool directSky = false;
    uint oidx = pix.y * Resolution.x + pix.x;   //デノイザへの入力の書き込み位置


    //GBuffer作成
    RayDesc ray;
    ray.Origin = ro;
    ray.Direction = rd;
    ray.TMin = 1e-3;    //数値誤差対策に少し浮かせて発射するのがミソ
    ray.TMax = 1e+5;    //10万MMDは本家の遠方クリッピング面までの距離と同じ
    TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 1,0,0, ray, payload);
    if (payload.imo == Dayo::Inai) {
        Dayo::GBuffer1[pix] = -1;
        Dayo::GBuffer2[pix] = 0;
        Dayo::NormalDepth[pix] = float4(0,0,0,0);
    } else {
        Patch P = GetPatch(payload.imo, payload.iface, payload.st);
        Dayo::GBuffer1[pix] = uint2(payload.imo, payload.iface);
        Dayo::GBuffer2[pix] = payload.st;
        Dayo::NormalDepth[pix] = float4(P.TBN[2],YRZ::Transform(P.pos, Dayo::ViewMatrix).z);
    }

    for (int i=0; i<nRays; i++) {
        ray.Origin = ro;
        ray.Direction = rd;
        TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 0,0,0, ray, payload);

        if (payload.imo == Dayo::Inai) {
            //何も当たらなかったので空を仰ぐ
            Lt += FetchSkybox(rd) * beta;
            directSky = (i==0);
            if (i==0) {
                //デノイザとGBufferへの出力
                Dayo::OIDNBuf[oidx].normal = 0;
                Dayo::OIDNBuf[oidx].albedo = 0;
            }
            break;
        } else {
            //当たったので反射先を追う
            Patch P = GetPatch(payload.imo, payload.iface, payload.st);
            float pdf;
            ro = P.pos;
            rd = YRZ::SampleCosWeightedHemisphere(P.TBN, RNG.xy, pdf);
            float3 brdf = P.m.albedo / PI;
            float geom = abs(dot(P.TBN[2],rd));
            beta *= geom * brdf / pdf;
            if (i==0) {
                //デノイザとGBufferへの出力を作る
                float w = 1.0/(iSample+1);
                Dayo::OIDNBuf[oidx].albedo = lerp(Dayo::OIDNBuf[oidx].albedo, P.m.albedo, w);
                Dayo::OIDNBuf[oidx].normal = normalize(lerp(Dayo::OIDNBuf[oidx].normal, P.TBN[2], w));
            }
        }
    }

    float4 cur = float4(Lt,directSky ? 0 : 1);
    Dayo::RTOutput[pix] = lerp(Dayo::RTOutput[pix], cur, 1.0/(iSample+1));
    Dayo::OIDNBuf[oidx].color = Dayo::RTOutput[pix];
}

}
```
### GBufferのための追加のレイトレ
まず、GBufferの方から説明しましょう。これは、`Dayo::GBuffer1,2`と`Dayo::NormalDepth`への出力として行います。どちらもRWTexture2Dという読み書き可能な2Dテクスチャです。

ポストプロセス編でちょっと取り上げたGBufferですが、レンダラが中身を作ってたんです。
これは、基本的にはレイが最初にヒットした場所での情報を書き込めばOKです。forループの前にもう一回`TraceRay()`を撃っています。ほぼコピペなんですけど

```c
    //GBuffer作成
    RayDesc ray;
    ray.Origin = ro;
    ray.Direction = rd;
    ray.TMin = 1e-3;    //数値誤差対策に少し浮かせて発射するのがミソ
    ray.TMax = 1e+5;    //10万MMDは本家の遠方クリッピング面までの距離と同じ
    TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 1,0,0, ray, payload);
    if (payload.imo == Dayo::Inai) {
        Dayo::GBuffer1[pix] = -1;
        Dayo::GBuffer2[pix] = 0;
        Dayo::NormalDepth[pix] = float4(0,0,0,0);
    } else {
        Patch P = GetPatch(payload.imo, payload.iface, payload.st);
        Dayo::GBuffer1[pix] = uint2(payload.imo, payload.iface);
        Dayo::GBuffer2[pix] = payload.st;
        Dayo::NormalDepth[pix] = float4(P.TBN[2],YRZ::Transform(P.pos, Dayo::ViewMatrix).z);
    }
```

で、よく見ると

```c
TraceRay(Dayo::TLAS, RAY_FLAG_NONE, 0xFF, 1,0,0, ray, payload);
```
として、第4引数が1になっています。これはhitGroupの何番のシェーダを使うかという指定です。今回はjson部でhitGroupを追加しましたが、ここで追加したhitGroupを使ってレイトレしてください、と変えている訳です。

続いてGBufferに書き込む値について説明しますと、skyboxに直撃した場合の「何も物体が書かれていない」事を示すデータは以下の通りです。

```c
Dayo::GBuffer1[pix] = Dayo::Inai;
Dayo::GBuffer2[pix] = 0;
Dayo::NormalDepth[pix] = float4(0,0,0,0);
```

`GBuffer1`の中身ですが、uint2になっていて
- r : モデル番号, `Dayo::Inai (-1)`でモデルは描かれていない事を示す。
- g : 面番号、モデル番号が`Dayo::Inai`の時は値は未定義です。

`GBuffer2`の中身は重心座標を書き込みますが、何もモデルが書かれていない場合は未定義です。書き込む必要自体無いのですが、一応0でクリアしています。

`NormalDepth`の中身もfloat4で
- rgb : ワールド座標系での法線
- a : 線形深度

となっています。線形深度というのは注目しているパッチに対するカメラのZ軸沿いの長さです。線形深度が0でクリアされていれば何も書かれていない画素である事が判別できます。

以上を踏まえて物体に当たった場合のコードを見てみましょう。

```c
Dayo::GBuffer1[pix] = uint2(payload.imo, payload.iface);
Dayo::GBuffer2[pix] = payload.st;
Dayo::NormalDepth[pix] = float4(P.TBN[2],YRZ::Transform(P.pos, Dayo::ViewMatrix).z);
```

`GBuffer1,2`の方はPayloadに書かれている内容そのままです。

`NormalDepth`の方は、rgb成分についてはパッチのワールド法線という事でわかりやすいですが、a成分は線形深度という事で、

1. YRZ::Transformでパッチの座標P.posをビュー座標系に変換する  
→ `mul(float4(P.pos,1), Dayo::ViewMatrix)`と同じです。
2. ビュー座標系でのP.posのz値が線形深度

と、それだけです。

### デノイザへの情報の作成
`Dayo::OIDNBuf`への書き込みが、デノイザへの入力の作成という事になります。`OIDNBuf`は構造体になっていて、どのメンバもfloat3型です。

- **color** デノイズ前の画像を入力します
- **albedo** 物体の反射率を入力します
- **normal** 物体の法線を入力します

物体の反射率と法線というのは、半透明な物体などが現れる場合はできれば「反射率の平均値」が良いそうです。法線の座標系はワールドでもビューでも良いとの事で、随分フレキシブルです。こちらもレイが最初にヒットする位置についての情報を元に作成していきます。

最初にレイがヒットするのはループ変数`i`が0の時の話ですから、その時の情報を元に`OIDNBuf`に書き込みをします。ただし、OIDNBufはテクスチャではなく、1次元のバッファ(RWStructuredBuffer)なので、書き込み位置の指定には`oidx`として1次元にならしたインデックスを使っています。

まず、レイがskyboxに直撃した時は、以下のように
```c
Dayo::OIDNBuf[oidx].normal = 0;
Dayo::OIDNBuf[oidx].albedo = 0;
```
クリアしてしまいます。こういう時はデノイザは「物体は置かれていないが何か背景画像が書かれているのであんまり真面目にデノイズしなくていいんじゃないか」と判断するようです。

次に、レイが何か物体に当たった時は、最初のヒットの時に
```c
Dayo::OIDNBuf[oidx].albedo = lerp(Dayo::OIDNBuf[oidx].albedo, P.m.albedo, w);
Dayo::OIDNBuf[oidx].normal = normalize(lerp(Dayo::OIDNBuf[oidx].normal, P.TBN[2], w));
```
対応する値を入れているだけ…ではあるんですが、サンプル間での平均を入れる事にしています。半透明の事も一応考えている訳です
albedoもnormalもシェーディングの時のように厳密な値が要求されるわけではなく、隣のピクセルとの間で大きな差が有ればあんまり乗り越えて画像をならすようなことをしない、という程度に緩い使われ方をしているようです。

colorの書き込みについては最後の所でRTOutputをコピーするだけです。
```
float4 cur = float4(Lt,directSky ? 0 : 1);
Dayo::RTOutput[pix] = lerp(Dayo::RTOutput[pix], cur, 1.0/(iSample+1));
Dayo::OIDNBuf[oidx].color = Dayo::RTOutput[pix];
```

以上でデノイザをONにするとノイズの取れた綺麗な出力をより手早く楽しめるようになったと思います。どうでしょう？なかなか悪くないんじゃないですか？

## 以上！
長かったレンダラ作成チュートリアルもこれで終了です。

改良の余地は無限にありますし、レイトレーシングを使わずにライタライザを使うレンダラとして実装する、というまた違う道もあります。

レンダラを作る上でほんとうに大事なのは、理屈の難しいものを作る事でも見た目のいいものを作る事でも軽く動作する物を作る事でも無く以下のシンプルなルールです

1. `MikuMikuDayo/renderer`フォルダに`.fxdayo`ファイルを置いてMikuMikuDayoを起動するとレンダラが選択・起動できるようになっているので、選択されればちゃんと動く
1. 輝度情報はガンマ補正前の**sRGBリニア色空間**での値を出力する
1. 背景だけ描画されているピクセルはα=0で出力する
1. モデルが描画されているピクセルはそれなりに出力する
1. デノイザ(`OIDNBuf`)とGBuffer(`GBuffer1,2`,`NormalDepth`)のためのデータを出力する
1. GBufferにはα値が`Dayo::AlphaThreshold`以上の一番手前にあるポリゴンについてのデータを載せる。

これさえ守れていれば**ちゃんとしたレンダラ**ダヨー