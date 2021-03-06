Title: Three.jsでゲームを作る
Description: THREE.jsでゲームを作る
Category: 解決策
TOC: ゲーム作りを始めよう

three.jsを使ってゲームを作りたいと思っている人は多いと思います。
この記事はどのようにして始めればいいのか、アイデアを与えてくれる事でしょう。

少なくとも私がこの記事を書いている時点では、このサイトで一番長い記事になると思います。
今回のコードは非常にオーバーエンジニアリングの可能性がありますが、
新しい機能を追加する度に問題にぶつかり、他のゲーム開発でよく使う解決策が必要でした。
つまり、それぞれの解決策が重要だと思うのでその理由を解説します。

あなたのゲームが小さければ小さいほど、ここで紹介されている解決策のいくつかは必要ないかもしれません。
今回のコードはかなり小さなゲームですが、3Dキャラクターの複雑さを考えると2Dキャラクターよりも多くの解決策が必要かもしれません。

例えば2Dでパックマンを作る場合、パックマンが角を曲がる時は即座に90度で曲がります。
中間のステップはありません。
しかし、3Dゲームの場合はいくつかのフレームにわたって回転するキャラクターが必要です。
その単純な変更はとても複雑で異なる解決策が必要です。

今回のコードの大部分で、**three.jsはゲームエンジンではないこと**に注意して下さい。
Three.jsは3Dライブラリです。
[シーングラフ](threejs-scenegraph.html)と3Dオブジェクトを表示する機能を提供します。
しかし、ゲームを作るための必要な機能は提供されていません。
例えばコリジョン、物理エンジン、入力システム、パスファインディングなどはないです。
そういうものは自分で用意する必要があります。

今回紹介するシンプルで *未完成* なゲーム風なものを作るためにかなりのコードが必要でした。
私が過剰に設計した可能性もあり、もっとシンプルな解決策もありますが、十分なコードを書いていなかったような気がします。

ここでのアイデアの多くは[Unity](https://unity.com)の影響を大きく受けています。
あなたがUnityに慣れていなくてもおそらく問題ありません。
このアイデアを使ったゲームが1000本以上出荷されているのでそれを持ち出しただけです。

まずはthree.jsの部分から始めましょう。ゲーム用の3Dモデルをロードする必要があります。

[opengameart.org](https://opengameart.org)で[quaternius](https://opengameart.org/users/quaternius)さんの[アニメーションする騎士モデル](https://opengameart.org/content/lowpoly-animated-knight)を見つけました。

<div class="threejs_center"><img src="resources/images/knight.jpg" style="width: 375px;"></div>

[quaternius](https://opengameart.org/users/quaternius)さんは[アニメーションする動物たち](https://opengameart.org/content/lowpoly-animated-farm-animal-pack)も作ってました。

<div class="threejs_center"><img src="resources/images/animals.jpg" style="width: 606px;"></div>

手始めに良さそうな3Dモデルなのでロードしてみます。

3Dモデルのロードは、以前に[glTFファイルの読み込み](threejs-load-gltf.html)で説明しました。
以前の説明と違うのは複数モデルをロードする必要があり、全てのモデルをロードしないとゲームを開始できません。

幸いな事にthree.jsでは、この目的のためだけに `LoadingManager` が提供されてます。
`LoadingManager` を作成し、それを他のローダーに渡します。
`LoadingManager`には、コールバックをアタッチできる[`onProgress`](LoadingManager.onProgress)と[`onLoad`](LoadingManager.onLoad)の両方のプロパティが用意されています。
全てのファイルのロード後に[`onLoad`](LoadingManager.onLoad)コールバックが呼び出されます。
それぞれのファイルがロード後に[`onProgress`](LoadingManager.onProgress) コールバックが呼び出され、ローディングの進捗状況を表示できます。

[glTFファイルの読み込み](threejs-load-gltf.html)のコードからシーンのフレーミングに関するコードを全て削除します。
そして、次のコードを追加して全モデルを読み込むようにしました。

```js
const manager = new THREE.LoadingManager();
manager.onLoad = init;
const models = {
  pig:    { url: 'resources/models/animals/Pig.gltf' },
  cow:    { url: 'resources/models/animals/Cow.gltf' },
  llama:  { url: 'resources/models/animals/Llama.gltf' },
  pug:    { url: 'resources/models/animals/Pug.gltf' },
  sheep:  { url: 'resources/models/animals/Sheep.gltf' },
  zebra:  { url: 'resources/models/animals/Zebra.gltf' },
  horse:  { url: 'resources/models/animals/Horse.gltf' },
  knight: { url: 'resources/models/knight/KnightCharacter.gltf' },
};
{
  const gltfLoader = new GLTFLoader(manager);
  for (const model of Object.values(models)) {
    gltfLoader.load(model.url, (gltf) => {
      model.gltf = gltf;
    });
  }
}

function init() {
  // TBD
}
```

このコードは全てのモデルをロードし `LoadingManager` が `init` を呼び出します。
モデルにアクセスするための `models` オブジェクトは後で使います。
そのため、`GLTFLoader` コールバックでロードされたデータをmodel.gltfにアタッチしておきます。

アニメーション付きの全モデルで約6.6MBです。
かなり大きなダウンロード量です。
サーバーが圧縮をサポートしていると仮定して（このサーバーはサイトが実行されている場合)、約1.4MBに圧縮できます。
6.6MBよりは確実に良いです、まだまだ微々たるデータですが。
ローディングのプログレスバーを追加し、ユーザーがどれくらい待つのか把握できると良いですね。

そこで[`onProgress`](LoadingManager.onProgress)コールバックを追加してみましょう。
これは3つの引数で呼び出され、最後に読み込まれたオブジェクトの `url`、読み込まれたアイテム数、読み込まれたアイテム総数を指定します。

ローディングバーのためのHTMLを追加しましょう。

```html
<body>
  <canvas id="c"></canvas>
+  <div id="loading">
+    <div>
+      <div>...loading...</div>
+      <div class="progress"><div id="progressbar"></div></div>
+    </div>
+  </div>
</body>
```

プログレスバーの幅を0〜100%に設定して進捗状況を表示できます。
あとはコールバックで設定するだけです。

```js
const manager = new THREE.LoadingManager();
manager.onLoad = init;

+const progressbarElem = document.querySelector('#progressbar');
+manager.onProgress = (url, itemsLoaded, itemsTotal) => {
+  progressbarElem.style.width = `${itemsLoaded / itemsTotal * 100 | 0}%`;
+};
```

全てのモデルロード後に `init` 呼び出されるようにし、`#loading` のDOM要素を隠してプログレスバーをオフします。

```js
function init() {
+  // hide the loading bar
+  const loadingElem = document.querySelector('#loading');
+  loadingElem.style.display = 'none';
}
```

以下はプログレスバーのCSSです。
`#loading` と `<div>` をページのフルサイズにして子要素を中央に配置します。
プログレスバーを格納するために `.progress` という領域を作ります。
また、プログレスバーに斜めストライプのCSSアニメーションを追加しています。

```css
#loading {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  font-size: xx-large;
  font-family: sans-serif;
}
#loading>div>div {
  padding: 2px;
}
.progress {
  width: 50vw;
  border: 1px solid black;
}
#progressbar {
  width: 0;
  transition: width ease-out .5s;
  height: 1em;
  background-color: #888;
  background-image: linear-gradient(
    -45deg, 
    rgba(255, 255, 255, .5) 25%, 
    transparent 25%, 
    transparent 50%, 
    rgba(255, 255, 255, .5) 50%, 
    rgba(255, 255, 255, .5) 75%, 
    transparent 75%, 
    transparent
  );
  background-size: 50px 50px;
  animation: progressanim 2s linear infinite;
}

@keyframes progressanim {
  0% {
    background-position: 50px 50px;
  }
  100% {
    background-position: 0 0;
  }
}
```

プログレスバーができたのでモデルを処理してみましょう。
モデルにはアニメーションがあり、アニメーションにアクセスできるようにしたいです。
アニメーションはデフォルトでは配列に格納されていますが、名前を付けて簡単にアクセスできるようにしましょう。
モデルごとに `animations` プロパティを設定します。
アニメーションにユニークな名前を付ける必要がある事に注意して下さい。

```js
+function prepModelsAndAnimations() {
+  Object.values(models).forEach(model => {
+    const animsByName = {};
+    model.gltf.animations.forEach((clip) => {
+      animsByName[clip.name] = clip;
+    });
+    model.animations = animsByName;
+  });
+}

function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

+  prepModelsAndAnimations();
}
```

アニメーション付きのモデルを表示してみましょう。

[前回のglTFファイルを読み込む例](threejs-load-gltf.html)とは異なり、今回は各モデルのインスタンスを表示できるようにしたいです。
そのためには[glTFを読み込む](threejs-load-gltf.html)のように読み込んだgltfシーンを直接追加するのではなく、
代わりにクローンを作成したいです（特にスキニングアニメーションするキャラクターに対して）。
幸いな事にこれを行うユーティリティ関数 `SkeletonUtils.clone` があります。
まず最初にこのUtilsをimportしておきます。

```js
import * as THREE from './resources/three/r122/build/three.module.js';
import {OrbitControls} from './resources/threejs/r122/examples/jsm/controls/OrbitControls.js';
import {GLTFLoader} from './resources/threejs/r122/examples/jsm/loaders/GLTFLoader.js';
+import {SkeletonUtils} from './resources/threejs/r122/examples/jsm/utils/SkeletonUtils.js';
```

そして、ロードしたモデルをクローンできます。

```js
function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();

+  Object.values(models).forEach((model, ndx) => {
+    const clonedScene = SkeletonUtils.clone(model.gltf.scene);
+    const root = new THREE.Object3D();
+    root.add(clonedScene);
+    scene.add(root);
+    root.position.x = (ndx - 3) * 3;
+  });
}
```

上記のように各モデルごとにロードした `gltf.scene` をクローンし `Object3D` を親にします。
別のオブジェクトを親にするのは、アニメーション再生時にロードされたシーンのノードにアニメーションの位置を適用するためです。

アニメーションを再生するには、クローンした各モデルに `AnimationMixer` が必要です。
`AnimationMixer` には1つ以上の `AnimationAction` が含まれます。
`AnimationAction` は `AnimationClip` を参照します。
複数の `AnimationAction` には再生したり、別のアクションに繋いだり、アクション間でクロスフェードするための設定が用意されています。
最初の `AnimationClip` を取得し、アクションを作成しましょう。
デフォルトでは、アクションクリップは永遠にループ再生になっています。

```js
+const mixers = [];

function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();

  Object.values(models).forEach((model, ndx) => {
    const clonedScene = SkeletonUtils.clone(model.gltf.scene);
    const root = new THREE.Object3D();
    root.add(clonedScene);
    scene.add(root);
    root.position.x = (ndx - 3) * 3;

+    const mixer = new THREE.AnimationMixer(clonedScene);
+    const firstClip = Object.values(model.animations)[0];
+    const action = mixer.clipAction(firstClip);
+    action.play();
+    mixers.push(mixer);
  });
}
```

アクションを開始するために[`play`](AnimationAction.play)を呼び出し、全ての `AnimationMixers` を `mixers` という配列に格納しました。
次にレンダーループ内で各 `AnimationMixer` を更新するため、最後のフレームからの時間を計算して `AnimationMixer.update` に渡します。

```js
+let then = 0;
function render(now) {
+  now *= 0.001;  // convert to seconds
+  const deltaTime = now - then;
+  then = now;

  if (resizeRendererToDisplaySize(renderer)) {
    const canvas = renderer.domElement;
    camera.aspect = canvas.clientWidth / canvas.clientHeight;
    camera.updateProjectionMatrix();
  }

+  for (const mixer of mixers) {
+    mixer.update(deltaTime);
+  }

  renderer.render(scene, camera);

  requestAnimationFrame(render);
}
```

これで各モデルをロードし、最初のアニメーションを再生できます。

{{{example url="../threejs-game-load-models.html"}}}

全てのアニメーションを確認できるようにしましょう。
全てのクリップをアクションとして追加し、1度に1つだけ有効にします。

```js
-const mixers = [];
+const mixerInfos = [];

function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();

  Object.values(models).forEach((model, ndx) => {
    const clonedScene = SkeletonUtils.clone(model.gltf.scene);
    const root = new THREE.Object3D();
    root.add(clonedScene);
    scene.add(root);
    root.position.x = (ndx - 3) * 3;

    const mixer = new THREE.AnimationMixer(clonedScene);
-    const firstClip = Object.values(model.animations)[0];
-    const action = mixer.clipAction(firstClip);
-    action.play();
-    mixers.push(mixer);
+    const actions = Object.values(model.animations).map((clip) => {
+      return mixer.clipAction(clip);
+    });
+    const mixerInfo = {
+      mixer,
+      actions,
+      actionNdx: -1,
+    };
+    mixerInfos.push(mixerInfo);
+    playNextAction(mixerInfo);
  });
}

+function playNextAction(mixerInfo) {
+  const {actions, actionNdx} = mixerInfo;
+  const nextActionNdx = (actionNdx + 1) % actions.length;
+  mixerInfo.actionNdx = nextActionNdx;
+  actions.forEach((action, ndx) => {
+    const enabled = ndx === nextActionNdx;
+    action.enabled = enabled;
+    if (enabled) {
+      action.play();
+    }
+  });
+}
```

上記のコードは各 `AnimationClip` に対応する `AnimationAction` の配列を作成しています。
オブジェクトの配列 `mixerInfos` を作成し、`AnimationMixer` と各モデルの全ての `AnimationAction` を参照します。
次に `playNextAction` を呼び出し、mixerの1つのアクションを除く全てのアクションを `enabled` します。

この新しい配列もレンダーループで更新が必要です。

```js
-for (const mixer of mixers) {
+for (const {mixer} of mixerInfos) {
  mixer.update(deltaTime);
}
```

1～8のキーを押すと、各モデルの次のアニメーションを再生するようにしてみましょう。

```js
window.addEventListener('keydown', (e) => {
  const mixerInfo = mixerInfos[e.keyCode - 49];
  if (!mixerInfo) {
    return;
  }
  playNextAction(mixerInfo);
});
```

これでexampleをクリックしてキー1〜8を押して、各モデルのアニメーションが循環できるようになりました。

{{{example url="../threejs-game-check-animations.html"}}}

この記事のthree.js部分のまとめは間違いなくこれです。
複数ファイルの読み込み、スキニングモデルのクローン作成、アニメーションの再生などを行いました。
実際のゲームでは `AnimationAction` オブジェクトの操作をもっとしなければなりません。

ゲームのインフラ（基盤）を作り始めよう。

最近のゲーム作成によくあるパターンは[エンティティコンポーネントシステム](https://www.google.com/search?q=entity+component+system)です。
エンティティコンポーネントシステムでは、ゲーム内のオブジェクトは *コンポーネント* の束で構成された *エンティティ* と呼ばれます。
どのコンポーネントをアタッチする（使用する）かを決めてエンティティを構築します。
では、エンティティコンポーネントシステムを作ってみましょう。

エンティティを `GameObject` と呼ぶ事にします。
実質的にはコンポーネントを束ねた `Object3D` です。

```js
function removeArrayElement(array, element) {
  const ndx = array.indexOf(element);
  if (ndx >= 0) {
    array.splice(ndx, 1);
  }
}

class GameObject {
  constructor(parent, name) {
    this.name = name;
    this.components = [];
    this.transform = new THREE.Object3D();
    parent.add(this.transform);
  }
  addComponent(ComponentType, ...args) {
    const component = new ComponentType(this, ...args);
    this.components.push(component);
    return component;
  }
  removeComponent(component) {
    removeArrayElement(this.components, component);
  }
  getComponent(ComponentType) {
    return this.components.find(c => c instanceof ComponentType);
  }
  update() {
    for (const component of this.components) {
      component.update();
    }
  }
}
```

`GameObject.update` を呼び出すと、全てのコンポーネントに対して `update` を呼び出します。

デバッガで `GameObject` を見た時、識別できるようにそれぞれ個別の名前を付けました。

ちょっと不思議に思える事もあります。

コンポーネント作成は `GameObject.addComponent` を使います。
これが良いアイデアなのか悪いアイデアなのか、私にはよく分かりません。
ゲームオブジェクトの外にコンポーネントが存在するのは意味がないので、
コンポーネント作成時に自動的にコンポーネントがゲームオブジェクトに追加され、ゲームオブジェクトがコンポーネントのコンストラクタに渡されるようになれば良いと考えました。
つまり、コンポーネントを追加するには次のようにします。

```js
const gameObject = new GameObject(scene, 'foo');
gameObject.addComponent(TypeOfComponent);
```

私が上記のようにしなかったら、あなたは代わりに以下のようにするでしょう。

```js
const gameObject = new GameObject(scene, 'foo');
const component = new TypeOfComponent(gameObject);
gameObject.addComponent(component);
```

最初の方法が短くて自動化されていて良いか、それとも普通じゃないように見えて悪いか私には分からないです。

`GameObject.getComponent` はタイプによってコンポーネントを探します。
これは1つのゲームオブジェクトに同じタイプの2つのコンポーネントを持てない事を意味しています。

あるコンポーネントが別のコンポーネントを検索するのが一般的です。
検索する際にはタイプ別に一致していなければならず、そうでなければ間違ったものを取得する可能性があります。
代わりに各コンポーネントに名前を付けて、名前で検索できます。
その方が同じタイプのコンポーネントを複数持つ事ができるという点では柔軟性がありますが、面倒くさくなります。
繰り返しになりますがどちらが良いのかは分かりません。

コンポーネント自体を見てみましょう。これが基底クラスです。

```js
// Base for all components
class Component {
  constructor(gameObject) {
    this.gameObject = gameObject;
  }
  update() {
  }
}
```

コンポーネントには基底クラスが必要ですか？
JavaScriptは厳密に型付けされた言語とは異なり、効果的には基底クラスを持たず、
コンストラクタの第1引数が常にコンポーネントのゲームオブジェクトである事を知っていれば、何をしたいかは各コンポーネントに任せます。
ゲームオブジェクトを気にしなければ保存しません。
なんとなく、この共通ベースで良い気がするんですけどね。
これはコンポーネントへの参照があれば、その親のゲームオブジェクトを見つける事ができます。
また、その親から他のコンポーネントを簡単に調べたり、transformを見る事ができます。

ゲームオブジェクトを管理するには、何らかのゲームオブジェクトマネージャーが必要です。
ゲームオブジェクトの配列を持てば良いと思うかもしれませんが、
実際のゲームではゲームオブジェクトのコンポーネントは、実行時に他のゲームオブジェクト追加または削除するかもしれません。
例えば銃のゲームオブジェクトは、銃が発射される度に弾丸のゲームオブジェクトを追加するかもしれません。
モンスターのゲームオブジェクトは、殺された場合に自身を削除するかもしれません。
そうすると、次のようなコードがあるかもしれないという問題が発生します。

```js
for (const gameObject of globalArrayOfGameObjects) {
  gameObject.update();
}
```

上記のコードはループの途中でコンポーネントの `update` 関数でゲームオブジェクトが
`globalArrayOfGameObjects` に追加や削除された場合に失敗したり、予期しない事が起こる可能性があります。

この問題を防ぐためにはもう少し安全なコードが必要です。ここに1つの試みがあります。

```js
class SafeArray {
  constructor() {
    this.array = [];
    this.addQueue = [];
    this.removeQueue = new Set();
  }
  get isEmpty() {
    return this.addQueue.length + this.array.length > 0;
  }
  add(element) {
    this.addQueue.push(element);
  }
  remove(element) {
    this.removeQueue.add(element);
  }
  forEach(fn) {
    this._addQueued();
    this._removeQueued();
    for (const element of this.array) {
      if (this.removeQueue.has(element)) {
        continue;
      }
      fn(element);
    }
    this._removeQueued();
  }
  _addQueued() {
    if (this.addQueue.length) {
      this.array.splice(this.array.length, 0, ...this.addQueue);
      this.addQueue = [];
    }
  }
  _removeQueued() {
    if (this.removeQueue.size) {
      this.array = this.array.filter(element => !this.removeQueue.has(element));
      this.removeQueue.clear();
    }
  }
}
```

上記の `SafeArray` クラスは要素を追加や削除できますが、イテレーター処理中は配列自体をいじる事はありません。
代わりに新しい要素は `addQueue` 配列に追加され、`removeQueue` 配列に削除された要素はループの外で追加または削除されます。

これを使い、ゲームオブジェクトを管理するクラスを作ります。

```js
class GameObjectManager {
  constructor() {
    this.gameObjects = new SafeArray();
  }
  createGameObject(parent, name) {
    const gameObject = new GameObject(parent, name);
    this.gameObjects.add(gameObject);
    return gameObject;
  }
  removeGameObject(gameObject) {
    this.gameObjects.remove(gameObject);
  }
  update() {
    this.gameObjects.forEach(gameObject => gameObject.update());
  }
}
```

これで最初のコンポーネントを作ってみましょう。
このコンポーネントは、先ほど作成したようなスキニングされたthree.jsオブジェクトを管理します。
シンプルなコードにするために、再生するアニメーションの名前を取得し再生する `setAnimation` というメソッドを1つ持ちます。

```js
class SkinInstance extends Component {
  constructor(gameObject, model) {
    super(gameObject);
    this.model = model;
    this.animRoot = SkeletonUtils.clone(this.model.gltf.scene);
    this.mixer = new THREE.AnimationMixer(this.animRoot);
    gameObject.transform.add(this.animRoot);
    this.actions = {};
  }
  setAnimation(animName) {
    const clip = this.model.animations[animName];
    // turn off all current actions
    for (const action of Object.values(this.actions)) {
      action.enabled = false;
    }
    // get or create existing action for clip
    const action = this.mixer.clipAction(clip);
    action.enabled = true;
    action.reset();
    action.play();
    this.actions[animName] = action;
  }
  update() {
    this.mixer.update(globals.deltaTime);
  }
}
```

基本的には先ほどのコードと同じで、ロードしたシーンをクローンして `AnimationMixer` を設定してます。
`setAnimation` は特定の `AnimationClip` に対して `AnimationAction` を追加します。
もし1つも存在しない場合、既存のアクションを全て無効にします。

このコードは `globals.deltaTime` を参照してます。
グローバルオブジェクトを作成しましょう。

```js
const globals = {
  time: 0,
  deltaTime: 0,
};
```

それをrenderループで更新します。

```js
let then = 0;
function render(now) {
  // convert to seconds
  globals.time = now * 0.001;
  // make sure delta time isn't too big.
  globals.deltaTime = Math.min(globals.time - then, 1 / 20);
  then = globals.time;
```

上記の `deltaTime` が1/20秒以下のチェックは、そうしないとブラウザのタブを非アクティブにすると `deltaTime` の値が大きくなるからです。
数秒 or 数分の間はそれを隠し、タブがアクティブ時に `deltaTime` が巨大になり、ゲームの世界でキャラクターをテレポートするかもしれません。

```js
position += velocity * deltaTime;
```

`deltaTime` の最大値を制限し、この問題を防ぐ事ができます。

では、プレイヤー用のコンポーネントを作ってみましょう。

```js
class Player extends Component {
  constructor(gameObject) {
    super(gameObject);
    const model = models.knight;
    this.skinInstance = gameObject.addComponent(SkinInstance, model);
    this.skinInstance.setAnimation('Run');
  }
}
```

プレイヤーは `setAnimation` を `Run'` で呼び出します。
利用可能なアニメーションを把握するために、前のexampleを修正してアニメーション名をコンソール出力しました。

```js
function prepModelsAndAnimations() {
  Object.values(models).forEach(model => {
+    console.log('------->:', model.url);
    const animsByName = {};
    model.gltf.animations.forEach((clip) => {
      animsByName[clip.name] = clip;
+      console.log('  ', clip.name);
    });
    model.animations = animsByName;
  });
}
```

これを実行すると[JavaScriptコンソール](https://developers.google.com/web/tools/chrome-devtools/console/javascript)に以下のリストが表示されます。

```
 ------->:  resources/models/animals/Pig.gltf
    Idle
    Death
    WalkSlow
    Jump
    Walk
 ------->:  resources/models/animals/Cow.gltf
    Walk
    Jump
    WalkSlow
    Death
    Idle
 ------->:  resources/models/animals/Llama.gltf
    Jump
    Idle
    Walk
    Death
    WalkSlow
 ------->:  resources/models/animals/Pug.gltf
    Jump
    Walk
    Idle
    WalkSlow
    Death
 ------->:  resources/models/animals/Sheep.gltf
    WalkSlow
    Death
    Jump
    Walk
    Idle
 ------->:  resources/models/animals/Zebra.gltf
    Jump
    Walk
    Death
    WalkSlow
    Idle
 ------->:  resources/models/animals/Horse.gltf
    Jump
    WalkSlow
    Death
    Walk
    Idle
 ------->:  resources/models/knight/KnightCharacter.gltf
    Run_swordRight
    Run
    Idle_swordLeft
    Roll_sword
    Idle
    Run_swordAttack
```

幸いな事に全ての動物のアニメーション名が一致しているのであとで便利になります。
今の所はプレイヤーが `Run` というアニメーションを持っている事だけ注意します。

これらのコンポーネントを使いましょう。
更新されたinit関数は以下の通りです。
`GameObject` を作成し、そこに `Player` コンポーネントを追加します。

```js
const globals = {
  time: 0,
  deltaTime: 0,
};
+const gameObjectManager = new GameObjectManager();

function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();

+  {
+    const gameObject = gameObjectManager.createGameObject(scene, 'player');
+    gameObject.addComponent(Player);
+  }
}
```

そして、renderループの中で `gameObjectManager.update` を呼び出します。

```js
let then = 0;
function render(now) {
  // convert to seconds
  globals.time = now * 0.001;
  // make sure delta time isn't too big.
  globals.deltaTime = Math.min(globals.time - then, 1 / 20);
  then = globals.time;

  if (resizeRendererToDisplaySize(renderer)) {
    const canvas = renderer.domElement;
    camera.aspect = canvas.clientWidth / canvas.clientHeight;
    camera.updateProjectionMatrix();
  }

-  for (const {mixer} of mixerInfos) {
-    mixer.update(deltaTime);
-  }
+  gameObjectManager.update();

  renderer.render(scene, camera);

  requestAnimationFrame(render);
}
```

それを実行するとシングルプレイヤーが得られます。

{{{example url="../threejs-game-just-player.html"}}}

エンティティコンポーネントシステムのためだけに多くのコードがありましたが、ほとんどのゲームで必要とされる基盤です。

入力システムを追加してみよう。
キーを直接読み込むのではなく、コードの他の部分が `left` や `right` をチェックできるようなクラスを作ります。
以下のように `left` や `right` などの複数の入力方法を割り当てできます。
まずはキーだけから始めましょう。

```js
// Keeps the state of keys/buttons
//
// You can check
//
//   inputManager.keys.left.down
//
// to see if the left key is currently held down
// and you can check
//
//   inputManager.keys.left.justPressed
//
// To see if the left key was pressed this frame
//
// Keys are 'left', 'right', 'a', 'b', 'up', 'down'
class InputManager {
  constructor() {
    this.keys = {};
    const keyMap = new Map();

    const setKey = (keyName, pressed) => {
      const keyState = this.keys[keyName];
      keyState.justPressed = pressed && !keyState.down;
      keyState.down = pressed;
    };

    const addKey = (keyCode, name) => {
      this.keys[name] = { down: false, justPressed: false };
      keyMap.set(keyCode, name);
    };

    const setKeyFromKeyCode = (keyCode, pressed) => {
      const keyName = keyMap.get(keyCode);
      if (!keyName) {
        return;
      }
      setKey(keyName, pressed);
    };

    addKey(37, 'left');
    addKey(39, 'right');
    addKey(38, 'up');
    addKey(40, 'down');
    addKey(90, 'a');
    addKey(88, 'b');

    window.addEventListener('keydown', (e) => {
      setKeyFromKeyCode(e.keyCode, true);
    });
    window.addEventListener('keyup', (e) => {
      setKeyFromKeyCode(e.keyCode, false);
    });
  }
  update() {
    for (const keyState of Object.values(this.keys)) {
      if (keyState.justPressed) {
        keyState.justPressed = false;
      }
    }
  }
}
```

上記のコードではキーが上 or 下を追跡します。
例えば `inputManager.keys.left.down` のようなチェックで最近キーが押されたかチェックできます。
また、キーごとに `justPressed` プロパティを持っており、ユーザがキーを押したかどうかを確認できます。
例えばジャンプキーの場合、ボタンが押されているかどうかでなく、ユーザーが今それを押したかどうかを知りたいのです。

それでは `InputManager` のインスタンスを作成してみましょう。

```js
const globals = {
  time: 0,
  deltaTime: 0,
};
const gameObjectManager = new GameObjectManager();
+const inputManager = new InputManager();
```

renderループでそれを更新します。

```js
function render(now) {

  ...

  gameObjectManager.update();
+  inputManager.update();

  ...
}
```

これは `gameObjectManager.update` の後に呼ばれる必要があります。
そうしないとコンポーネントの `update` 内で `justPressed` がtrueになりません。

それを `Player` コンポーネントで使ってみましょう。

```js
+const kForward = new THREE.Vector3(0, 0, 1);
const globals = {
  time: 0,
  deltaTime: 0,
+  moveSpeed: 16,
};

class Player extends Component {
  constructor(gameObject) {
    super(gameObject);
    const model = models.knight;
    this.skinInstance = gameObject.addComponent(SkinInstance, model);
    this.skinInstance.setAnimation('Run');
+    this.turnSpeed = globals.moveSpeed / 4;
  }
+  update() {
+    const {deltaTime, moveSpeed} = globals;
+    const {transform} = this.gameObject;
+    const delta = (inputManager.keys.left.down  ?  1 : 0) +
+                  (inputManager.keys.right.down ? -1 : 0);
+    transform.rotation.y += this.turnSpeed * delta * deltaTime;
+    transform.translateOnAxis(kForward, moveSpeed * deltaTime);
+  }
}
```

上記のコードは `Object3D.transformOnAxis` でプレイヤーを前進させてます。
`Object3D.transformOnAxis` はローカル空間で動作するので、
問題のオブジェクトがシーンのルートにある場合にのみ動作し、
他の<a class="footnote" href="#parented" id="parented-backref">何か</a>の親になっている場合には動作しません。
また、グローバルな `moveSpeed` を追加し、移動速度に基づいて `turnSpeed` を設定しました。
ターン速度は、キャラクターが目標に見合って鋭くターンできる移動速度に基づいています。
`turnSpeed` が小さすぎると、キャラクターは目標を旋回しながらも決して目標に当たりません。
与えられた移動速度に必要なターン速度を計算するために、わざわざ計算をしていませんでした。推測しただけです。

これまでのコードでは動作しますが、プレイヤーが画面外に出てしまった場合にどこにいるのかを確認する方法がありません。
一定時間以上、画面外にいるとテレポートされて元の場所に戻るようにしましょう。
これを行うにはthree.jsの `Frustum` クラスを使い、点がカメラのビューの錐台内にいるかチェックします。
カメラから錐台を作る必要があります。
これをPlayerコンポーネントで行う事ができますが、他のオブジェクトもこれを使いたいかもしれないので、
錐台を管理するコンポーネントを持つ別のゲームオブジェクトを追加してみましょう。

```js
class CameraInfo extends Component {
  constructor(gameObject) {
    super(gameObject);
    this.projScreenMatrix = new THREE.Matrix4();
    this.frustum = new THREE.Frustum();
  }
  update() {
    const {camera} = globals;
    this.projScreenMatrix.multiplyMatrices(
        camera.projectionMatrix,
        camera.matrixWorldInverse);
    this.frustum.setFromProjectionMatrix(this.projScreenMatrix);
  }
}
```

init時に別のゲームオブジェクトを設定してみましょう。

```js
function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();

+  {
+    const gameObject = gameObjectManager.createGameObject(camera, 'camera');
+    globals.cameraInfo = gameObject.addComponent(CameraInfo);
+  }

  {
    const gameObject = gameObjectManager.createGameObject(scene, 'player');
    gameObject.addComponent(Player);
  }
}
```

これを `Player` コンポーネントで使う事ができるようになりました。

```js
class Player extends Component {
  constructor(gameObject) {
    super(gameObject);
    const model = models.knight;
    this.skinInstance = gameObject.addComponent(SkinInstance, model);
    this.skinInstance.setAnimation('Run');
    this.turnSpeed = globals.moveSpeed / 4;
+    this.offscreenTimer = 0;
+    this.maxTimeOffScreen = 3;
  }
  update() {
-    const {deltaTime, moveSpeed} = globals;
+    const {deltaTime, moveSpeed, cameraInfo} = globals;
    const {transform} = this.gameObject;
    const delta = (inputManager.keys.left.down  ?  1 : 0) +
                  (inputManager.keys.right.down ? -1 : 0);
    transform.rotation.y += this.turnSpeed * delta * deltaTime;
    transform.translateOnAxis(kForward, moveSpeed * deltaTime);

+    const {frustum} = cameraInfo;
+    if (frustum.containsPoint(transform.position)) {
+      this.offscreenTimer = 0;
+    } else {
+      this.offscreenTimer += deltaTime;
+      if (this.offscreenTimer >= this.maxTimeOffScreen) {
+        transform.position.set(0, 0, 0);
+      }
+    }
  }
}
```

試す前にもう1つ、モバイルのタッチスクリーン対応を追加しておきましょう。
まず、HTMLを追加してタッチしてみましょう。

```html
<body>
  <canvas id="c"></canvas>
+  <div id="ui">
+    <div id="left"><img src="resources/images/left.svg"></div>
+    <div style="flex: 0 0 40px;"></div>
+    <div id="right"><img src="resources/images/right.svg"></div>
+  </div>
  <div id="loading">
    <div>
      <div>...loading...</div>
      <div class="progress"><div id="progressbar"></div></div>
    </div>
  </div>
</body>
```

いくつかのCSSでスタイルを整えます。

```css
#ui {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-items: center;
  align-content: stretch;
}
#ui>div {
  display: flex;
  align-items: flex-end;
  flex: 1 1 auto;
}
.bright {
  filter: brightness(2);
}
#left {
  justify-content: flex-end;
}
#right {
  justify-content: flex-start;
}
#ui img {
  padding: 10px;
  width: 80px;
  height: 80px;
  display: block;
}
```

ここでの考え方は、ページ全体をカバーするdiv `#ui` が1つあります。
内部には2つのdiv、`#left` と `#right` があり、どちらもページ幅のほぼ半分、画面全体の高さがあります。
その間には40pxの区切りがあります。
ユーザーが指を左右にスライドさせた場合、`InputManager` の `keys.left` と `keys.right` を更新する必要があります。
これにより画面全体がタッチされてることに敏感になり、ただの小さな矢印よりも良いように思えました。

```js
class InputManager {
  constructor() {
    this.keys = {};
    const keyMap = new Map();

    const setKey = (keyName, pressed) => {
      const keyState = this.keys[keyName];
      keyState.justPressed = pressed && !keyState.down;
      keyState.down = pressed;
    };

    const addKey = (keyCode, name) => {
      this.keys[name] = { down: false, justPressed: false };
      keyMap.set(keyCode, name);
    };

    const setKeyFromKeyCode = (keyCode, pressed) => {
      const keyName = keyMap.get(keyCode);
      if (!keyName) {
        return;
      }
      setKey(keyName, pressed);
    };

    addKey(37, 'left');
    addKey(39, 'right');
    addKey(38, 'up');
    addKey(40, 'down');
    addKey(90, 'a');
    addKey(88, 'b');

    window.addEventListener('keydown', (e) => {
      setKeyFromKeyCode(e.keyCode, true);
    });
    window.addEventListener('keyup', (e) => {
      setKeyFromKeyCode(e.keyCode, false);
    });

+    const sides = [
+      { elem: document.querySelector('#left'),  key: 'left'  },
+      { elem: document.querySelector('#right'), key: 'right' },
+    ];
+
+    const clearKeys = () => {
+      for (const {key} of sides) {
+          setKey(key, false);
+      }
+    };
+
+    const checkSides = (e) => {
+      for (const {elem, key} of sides) {
+        let pressed = false;
+        const rect = elem.getBoundingClientRect();
+        for (const touch of e.touches) {
+          const x = touch.clientX;
+          const y = touch.clientY;
+          const inRect = x >= rect.left && x < rect.right &&
+                         y >= rect.top && y < rect.bottom;
+          if (inRect) {
+            pressed = true;
+          }
+        }
+        setKey(key, pressed);
+      }
+    };
+
+    const uiElem = document.querySelector('#ui');
+    uiElem.addEventListener('touchstart', (e) => {
+      e.preventDefault();
+      checkSides(e);
+    }, {passive: false});
+    uiElem.addEventListener('touchmove', (e) => {
+      e.preventDefault();  // prevent scroll
+      checkSides(e);
+    }, {passive: false});
+    uiElem.addEventListener('touchend', () => {
+      clearKeys();
+    });
+
+    function handleMouseMove(e) {
+      e.preventDefault();
+      checkSides({
+        touches: [e],
+      });
+    }
+
+    function handleMouseUp() {
+      clearKeys();
+      window.removeEventListener('mousemove', handleMouseMove, {passive: false});
+      window.removeEventListener('mouseup', handleMouseUp);
+    }
+
+    uiElem.addEventListener('mousedown', (e) => {
+      handleMouseMove(e);
+      window.addEventListener('mousemove', handleMouseMove);
+      window.addEventListener('mouseup', handleMouseUp);
+    }, {passive: false});
  }
  update() {
    for (const keyState of Object.values(this.keys)) {
      if (keyState.justPressed) {
        keyState.justPressed = false;
      }
    }
  }
}
```

そしてタッチスクリーン上で指で左右のカーソルキーやキャラクターを操作できるようになっています。

{{{example url="../threejs-game-player-input.html"}}}

理想的にはプレイヤーがカメラを移動させたり、オフスクリーン = プレイヤーの死亡のように画面外に出た場合に何か他の事をしたいですが、この記事が長くなりそうなので、真ん中にテレポートする事が最も簡単でした。

いくつかの動物を追加してみましょう。
まず、`Animal` コンポーネントを作る事で `Player` と同様に始められます。

```js
class Animal extends Component {
  constructor(gameObject, model) {
    super(gameObject);
    const skinInstance = gameObject.addComponent(SkinInstance, model);
    skinInstance.mixer.timeScale = globals.moveSpeed / 4;
    skinInstance.setAnimation('Idle');
  }
}
```

上記のコードでは `AnimationMixer.timeScale` を設定し、移動速度に対するアニメーションの再生速度を設定しています。
このように移動速度を調整すると、アニメーションの速度が速くなったり遅くなったりします。

最初にそれぞれの動物の種類を設定できます。

```js
function init() {
  // hide the loading bar
  const loadingElem = document.querySelector('#loading');
  loadingElem.style.display = 'none';

  prepModelsAndAnimations();
  {
    const gameObject = gameObjectManager.createGameObject(camera, 'camera');
    globals.cameraInfo = gameObject.addComponent(CameraInfo);
  }

  {
    const gameObject = gameObjectManager.createGameObject(scene, 'player');
    globals.player = gameObject.addComponent(Player);
    globals.congaLine = [gameObject];
  }

+  const animalModelNames = [
+    'pig',
+    'cow',
+    'llama',
+    'pug',
+    'sheep',
+    'zebra',
+    'horse',
+  ];
+  animalModelNames.forEach((name, ndx) => {
+    const gameObject = gameObjectManager.createGameObject(scene, name);
+    gameObject.addComponent(Animal, models[name]);
+    gameObject.transform.position.x = (ndx + 1) * 5;
+  });
}
```

そうすると動物たちが画面に立っているけど、何かしてもらいたいですね。

コンガのラインの中でプレイヤーの後を追わせてみましょう。
ただし、プレイヤーが十分に近づいた場合に限ります。
そのためにはいくつかの状態が必要です。

* アイドル:

  動物はプレイヤーが近づくまで待っています

* 終点待ち:

  動物はプレイヤーによってタグ付けされましたが、ラインの端に参加できるようにラインの端に来て動物を待つ必要があります。

* 終点へ行く:

  動物は追っている動物が現在いる場所の履歴を記録すると同時に、追っている動物がいた場所まで歩く必要があります。

* 尾行する:

  動物は追っている動物が以前いた場所に移動しながら、追っている動物がどこにいるかの履歴を記録しておく必要があります。

このように様々な状態に対応する方法があります。
一般的には[有限状態マシン](https://www.google.com/search?q=finite+state+machine)と
状態を管理するためのクラスを作成します。

つまり、それを作成しましょう。

```js
class FiniteStateMachine {
  constructor(states, initialState) {
    this.states = states;
    this.transition(initialState);
  }
  get state() {
    return this.currentState;
  }
  transition(state) {
    const oldState = this.states[this.currentState];
    if (oldState && oldState.exit) {
      oldState.exit.call(this);
    }
    this.currentState = state;
    const newState = this.states[state];
    if (newState.enter) {
      newState.enter.call(this);
    }
  }
  update() {
    const state = this.states[this.currentState];
    if (state.update) {
      state.update.call(this);
    }
  }
}
```

ここに簡単なクラスがあります。
たくさんの状態を持つオブジェクトを渡します。
それぞれの状態は `enter`、`update`、`exit` の3つのオプション関数で構成されています。
状態を切り替えるには、`FiniteStateMachine.transition` を呼び出し、新しい状態の名前を渡します。
現在の状態に `exit` 関数がある場合、それが呼び出されます。
新しい状態に `enter` 関数がある場合、それが呼び出されます。
最後に各フレームの `FiniteStateMachine.update` は現在の状態の `update` 関数を呼び出します。

動物の状態を管理するのに使いましょう。

```js
// Returns true of obj1 and obj2 are close
function isClose(obj1, obj1Radius, obj2, obj2Radius) {
  const minDist = obj1Radius + obj2Radius;
  const dist = obj1.position.distanceTo(obj2.position);
  return dist < minDist;
}

// keeps v between -min and +min
function minMagnitude(v, min) {
  return Math.abs(v) > min
      ? min * Math.sign(v)
      : v;
}

const aimTowardAndGetDistance = function() {
  const delta = new THREE.Vector3();

  return function aimTowardAndGetDistance(source, targetPos, maxTurn) {
    delta.subVectors(targetPos, source.position);
    // compute the direction we want to be facing
    const targetRot = Math.atan2(delta.x, delta.z) + Math.PI * 1.5;
    // rotate in the shortest direction
    const deltaRot = (targetRot - source.rotation.y + Math.PI * 1.5) % (Math.PI * 2) - Math.PI;
    // make sure we don't turn faster than maxTurn
    const deltaRotation = minMagnitude(deltaRot, maxTurn);
    // keep rotation between 0 and Math.PI * 2
    source.rotation.y = THREE.MathUtils.euclideanModulo(
        source.rotation.y + deltaRotation, Math.PI * 2);
    // return the distance to the target
    return delta.length();
  };
}();

class Animal extends Component {
  constructor(gameObject, model) {
    super(gameObject);
+    const hitRadius = model.size / 2;
    const skinInstance = gameObject.addComponent(SkinInstance, model);
    skinInstance.mixer.timeScale = globals.moveSpeed / 4;
+    const transform = gameObject.transform;
+    const playerTransform = globals.player.gameObject.transform;
+    const maxTurnSpeed = Math.PI * (globals.moveSpeed / 4);
+    const targetHistory = [];
+    let targetNdx = 0;
+
+    function addHistory() {
+      const targetGO = globals.congaLine[targetNdx];
+      const newTargetPos = new THREE.Vector3();
+      newTargetPos.copy(targetGO.transform.position);
+      targetHistory.push(newTargetPos);
+    }
+
+    this.fsm = new FiniteStateMachine({
+      idle: {
+        enter: () => {
+          skinInstance.setAnimation('Idle');
+        },
+        update: () => {
+          // check if player is near
+          if (isClose(transform, hitRadius, playerTransform, globals.playerRadius)) {
+            this.fsm.transition('waitForEnd');
+          }
+        },
+      },
+      waitForEnd: {
+        enter: () => {
+          skinInstance.setAnimation('Jump');
+        },
+        update: () => {
+          // get the gameObject at the end of the conga line
+          const lastGO = globals.congaLine[globals.congaLine.length - 1];
+          const deltaTurnSpeed = maxTurnSpeed * globals.deltaTime;
+          const targetPos = lastGO.transform.position;
+          aimTowardAndGetDistance(transform, targetPos, deltaTurnSpeed);
+          // check if last thing in conga line is near
+          if (isClose(transform, hitRadius, lastGO.transform, globals.playerRadius)) {
+            this.fsm.transition('goToLast');
+          }
+        },
+      },
+      goToLast: {
+        enter: () => {
+          // remember who we're following
+          targetNdx = globals.congaLine.length - 1;
+          // add ourselves to the conga line
+          globals.congaLine.push(gameObject);
+          skinInstance.setAnimation('Walk');
+        },
+        update: () => {
+          addHistory();
+          // walk to the oldest point in the history
+          const targetPos = targetHistory[0];
+          const maxVelocity = globals.moveSpeed * globals.deltaTime;
+          const deltaTurnSpeed = maxTurnSpeed * globals.deltaTime;
+          const distance = aimTowardAndGetDistance(transform, targetPos, deltaTurnSpeed);
+          const velocity = distance;
+          transform.translateOnAxis(kForward, Math.min(velocity, maxVelocity));
+          if (distance <= maxVelocity) {
+            this.fsm.transition('follow');
+          }
+        },
+      },
+      follow: {
+        update: () => {
+          addHistory();
+          // remove the oldest history and just put ourselves there.
+          const targetPos = targetHistory.shift();
+          transform.position.copy(targetPos);
+          const deltaTurnSpeed = maxTurnSpeed * globals.deltaTime;
+          aimTowardAndGetDistance(transform, targetHistory[0], deltaTurnSpeed);
+        },
+      },
+    }, 'idle');
+  }
+  update() {
+    this.fsm.update();
+  }
}
```

これは大きなコードの塊でしたが、上記で説明したような状態管理をしてくれます。
上手くいけばそれぞれの状態が遷移できて、現在はどの状態か明らかになるでしょう。

追加しなければならないものがいくつかあります。
動物が見つけられるようにプレイヤー自身をグローバルに追加し、プレイヤーの `GameObject` でコンガラインを開始する必要があります。

```js
function init() {

  ...

  {
    const gameObject = gameObjectManager.createGameObject(scene, 'player');
+    globals.player = gameObject.addComponent(Player);
+    globals.congaLine = [gameObject];
  }

}
```

また、各モデルのサイズを計算する必要があります。

```js
function prepModelsAndAnimations() {
+  const box = new THREE.Box3();
+  const size = new THREE.Vector3();
  Object.values(models).forEach(model => {
+    box.setFromObject(model.gltf.scene);
+    box.getSize(size);
+    model.size = size.length();
    const animsByName = {};
    model.gltf.animations.forEach((clip) => {
      animsByName[clip.name] = clip;
      // Should really fix this in .blend file
      if (clip.name === 'Walk') {
        clip.duration /= 2;
      }
    });
    model.animations = animsByName;
  });
}
```

そして、プレイヤーが動物達のサイズを記録する必要があります。

```js
class Player extends Component {
  constructor(gameObject) {
    super(gameObject);
    const model = models.knight;
+    globals.playerRadius = model.size / 2;
```

Thinking about it now it would probably have been smarter
for the animals to just target the head of the conga line
instead of the player specifically. Maybe I'll come back
and change that later.

When I first started this I used just one radius for all animals
but of course that was no good as the pug is much smaller than the horse.
So I added the difference sizes but I wanted to be able to visualize
things. To do that I made a `StatusDisplayHelper` component.

I uses a `PolarGridHelper` to draw a circle around each character
and it uses html elements to let each character show some status using
the techniques covered in [the article on aligning html elements to 3D](threejs-align-html-elements-to-3d.html).

First we need to add some HTML to host these elements

```html
<body>
  <canvas id="c"></canvas>
  <div id="ui">
    <div id="left"><img src="resources/images/left.svg"></div>
    <div style="flex: 0 0 40px;"></div>
    <div id="right"><img src="resources/images/right.svg"></div>
  </div>
  <div id="loading">
    <div>
      <div>...loading...</div>
      <div class="progress"><div id="progressbar"></div></div>
    </div>
  </div>
+  <div id="labels"></div>
</body>
```

And add some CSS for them

```css
#labels {
  position: absolute;  /* let us position ourself inside the container */
  left: 0;             /* make our position the top left of the container */
  top: 0;
  color: white;
  width: 100%;
  height: 100%;
  overflow: hidden;
  pointer-events: none;
}
#labels>div {
  position: absolute;  /* let us position them inside the container */
  left: 0;             /* make their default position the top left of the container */
  top: 0;
  font-size: large;
  font-family: monospace;
  user-select: none;   /* don't let the text get selected */
  text-shadow:         /* create a black outline */
    -1px -1px 0 #000,
     0   -1px 0 #000,
     1px -1px 0 #000,
     1px  0   0 #000,
     1px  1px 0 #000,
     0    1px 0 #000,
    -1px  1px 0 #000,
    -1px  0   0 #000;
}
```

Then here's the component

```js
const labelContainerElem = document.querySelector('#labels');

class StateDisplayHelper extends Component {
  constructor(gameObject, size) {
    super(gameObject);
    this.elem = document.createElement('div');
    labelContainerElem.appendChild(this.elem);
    this.pos = new THREE.Vector3();

    this.helper = new THREE.PolarGridHelper(size / 2, 1, 1, 16);
    gameObject.transform.add(this.helper);
  }
  setState(s) {
    this.elem.textContent = s;
  }
  setColor(cssColor) {
    this.elem.style.color = cssColor;
    this.helper.material.color.set(cssColor);
  }
  update() {
    const {pos} = this;
    const {transform} = this.gameObject;
    const {canvas} = globals;
    pos.copy(transform.position);

    // get the normalized screen coordinate of that position
    // x and y will be in the -1 to +1 range with x = -1 being
    // on the left and y = -1 being on the bottom
    pos.project(globals.camera);

    // convert the normalized position to CSS coordinates
    const x = (pos.x *  .5 + .5) * canvas.clientWidth;
    const y = (pos.y * -.5 + .5) * canvas.clientHeight;

    // move the elem to that position
    this.elem.style.transform = `translate(-50%, -50%) translate(${x}px,${y}px)`;
  }
}
```

And we can then add them to the animals like this

```js
class Animal extends Component {
  constructor(gameObject, model) {
    super(gameObject);
+    this.helper = gameObject.addComponent(StateDisplayHelper, model.size);

     ...

  }
  update() {
    this.fsm.update();
+    const dir = THREE.MathUtils.radToDeg(this.gameObject.transform.rotation.y);
+    this.helper.setState(`${this.fsm.state}:${dir.toFixed(0)}`);
  }
}
```

While we're at it lets make it so we can turn them on/off using dat.GUI like
we've used else where

```js
import * as THREE from './resources/three/r122/build/three.module.js';
import {OrbitControls} from './resources/threejs/r122/examples/jsm/controls/OrbitControls.js';
import {GLTFLoader} from './resources/threejs/r122/examples/jsm/loaders/GLTFLoader.js';
import {SkeletonUtils} from './resources/threejs/r122/examples/jsm/utils/SkeletonUtils.js';
+import {GUI} from '../3rdparty/dat.gui.module.js';
```

```js
+const gui = new GUI();
+gui.add(globals, 'debug').onChange(showHideDebugInfo);
+showHideDebugInfo();

const labelContainerElem = document.querySelector('#labels');
+function showHideDebugInfo() {
+  labelContainerElem.style.display = globals.debug ? '' : 'none';
+}
+showHideDebugInfo();

class StateDisplayHelper extends Component {

  ...

  update() {
+    this.helper.visible = globals.debug;
+    if (!globals.debug) {
+      return;
+    }

    ...
  }
}
```

And with that we get the kind of start of a game

{{{example url="../threejs-game-conga-line.html"}}}

Originally I set out to make a [snake game](https://www.google.com/search?q=snake+game)
where as you add animals to your line it gets harder because you need to avoid
crashing into them. I'd also have put some obstacles in the scene and maybe a fence or some
barrier around the perimeter.

Unfortunately the animals are long and thin. From above here's the zebra.

<div class="threejs_center"><img src="resources/images/zebra.png" style="width: 113px;"></div>

The code so far is using circle collisions which means if we had obstacles like a fence
then this would be considered a collision

<div class="threejs_center"><img src="resources/images/zebra-collisions.svg" style="width: 400px;"></div>

That's no good. Even animal to animal we'd have the same issue

I thought about writing a 2D rectangle to rectangle collision system but I
quickly realized it could really be a lot of code. Checking that 2 arbitrarily
oriented boxes overlap is not too much code and for our game with just a few
objects it might work but looking into it after a few objects you quickly start
needing to optimize the collision checking. First you might go through all
objects that can possibly collide with each other and check their bounding
spheres or bounding circles or their axially aligned bounding boxes. Once you
know which objects *might* be colliding then you need to do more work to check if
they are *actually* colliding. Often even checking the bounding spheres is too
much work and you need some kind of better spacial structure for the objects so
you can more quickly only check objects possibly near each other.

Then, once you write the code to check if 2 objects collide you generally want
to make a collision system rather than manually asking "do I collide with these
objects". A collision system emits events or calls callbacks in relation to
things colliding. The advantage is it can check all the collisions at once so no
objects get checked more than once where as if you manually call some "am I
colliding" function often objects will be checked more than once wasting time.

Making that collision system would probably not be more than 100-300 lines of
code for just checking arbitrarily oriented rectangles but it's still a ton more
code so it seemed best to leave it out.

Another solution would have been to try to find other characters that are
mostly circular from the top. Other humanoid characters for example instead
of animals in which case the circle checking might work animal to animal. 
It would not work animal to fence, well we'd have to add circle to rectangle
checking. I thought about making the fence a fence of bushes or poles, something
circular but then I'd need probably 120 to 200 of them to surround the play area
which would run into the optimization issues mentioned above.

These are reasons many games use an existing solution. Often these solutions
are part of a physics library. The physical library needs to know if objects
collide with each other so on top of providing physics they can also be used
to detect collision.

If you're looking for a solution some of the three.js examples use
[ammo.js](https://github.com/kripken/ammo.js/) so that might be one.

One other solution might have been to place the obstacles on a grid
and try to make it so each animal and the player just need to look at
the grid. While that would be performant I felt that's best left as an exercise
for the reader 😜

One more thing, many game systems have something called [*coroutines*](https://www.google.com/search?q=coroutines).
Coroutines are routines that can pause while running and continue later.

Let's make the main character emit musical notes like they are leading
the line by singing. There are many ways we could implement this but for now
let's do it using coroutines.

First, here's a class to manage coroutines

```js
function* waitSeconds(duration) {
  while (duration > 0) {
    duration -= globals.deltaTime;
    yield;
  }
}

class CoroutineRunner {
  constructor() {
    this.generatorStacks = [];
    this.addQueue = [];
    this.removeQueue = new Set();
  }
  isBusy() {
    return this.addQueue.length + this.generatorStacks.length > 0;
  }
  add(generator, delay = 0) {
    const genStack = [generator];
    if (delay) {
      genStack.push(waitSeconds(delay));
    }
    this.addQueue.push(genStack);
  }
  remove(generator) {
    this.removeQueue.add(generator);
  }
  update() {
    this._addQueued();
    this._removeQueued();
    for (const genStack of this.generatorStacks) {
      const main = genStack[0];
      // Handle if one coroutine removes another
      if (this.removeQueue.has(main)) {
        continue;
      }
      while (genStack.length) {
        const topGen = genStack[genStack.length - 1];
        const {value, done} = topGen.next();
        if (done) {
          if (genStack.length === 1) {
            this.removeQueue.add(topGen);
            break;
          }
          genStack.pop();
        } else if (value) {
          genStack.push(value);
        } else {
          break;
        }
      }
    }
    this._removeQueued();
  }
  _addQueued() {
    if (this.addQueue.length) {
      this.generatorStacks.splice(this.generatorStacks.length, 0, ...this.addQueue);
      this.addQueue = [];
    }
  }
  _removeQueued() {
    if (this.removeQueue.size) {
      this.generatorStacks = this.generatorStacks.filter(genStack => !this.removeQueue.has(genStack[0]));
      this.removeQueue.clear();
    }
  }
}
```

It does things similar to `SafeArray` to make sure that it's safe to add or remove
coroutines while other coroutines are running. It also handles nested coroutines.

To make a coroutine you make a [JavaScript generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*).
A generator function is preceded by the keyword `function*` (the asterisk is important!)

Generator functions can `yield`. For example

```js
function* countOTo9() {
  for (let i = 0; i < 10; ++i) {
    console.log(i);
    yield;
  }
}
```

If we added this function to the `CoroutineRunner` above it would print
out each number, 0 to 9, once per frame or rather once per time we called `runner.update`.

```js
const runner = new CoroutineRunner();
runner.add(count0To9);
while(runner.isBusy()) {
  runner.update();
}
```

Coroutines are removed automatically when they are finished. 
To remove a coroutine early, before it reaches the end you need to keep
a reference to its generator like this

```js
const gen = count0To9();
runner.add(gen);

// sometime later

runner.remove(gen);
```

In any case, in the player let's use a coroutine to emit a note every half second to 1 second

```js
class Player extends Component {
  constructor(gameObject) {

    ...

+    this.runner = new CoroutineRunner();
+
+    function* emitNotes() {
+      for (;;) {
+        yield waitSeconds(rand(0.5, 1));
+        const noteGO = gameObjectManager.createGameObject(scene, 'note');
+        noteGO.transform.position.copy(gameObject.transform.position);
+        noteGO.transform.position.y += 5;
+        noteGO.addComponent(Note);
+      }
+    }
+
+    this.runner.add(emitNotes());
  }
  update() {
+    this.runner.update();

  ...

  }
}

function rand(min, max) {
  if (max === undefined) {
    max = min;
    min = 0;
  }
  return Math.random() * (max - min) + min;
}
```

You can see we make a `CoroutineRunner` and we add an `emitNotes` coroutine.
That function will run forever, waiting 0.5 to 1 seconds and then creating a game object
with a `Note` component.

For the `Note` component first lets make a texture with a note on it and 
instead of loading a note image let's make one using a canvas like we covered in [the article on canvas textures](threejs-canvas-textures.html).

```js
function makeTextTexture(str) {
  const ctx = document.createElement('canvas').getContext('2d');
  ctx.canvas.width = 64;
  ctx.canvas.height = 64;
  ctx.font = '60px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillStyle = '#FFF';
  ctx.fillText(str, ctx.canvas.width / 2, ctx.canvas.height / 2);
  return new THREE.CanvasTexture(ctx.canvas);
}
const noteTexture = makeTextTexture('♪');
```

The texture we create above is white each means when we use it
we can set the material's color and get a note of any color.

Now that we have a noteTexture here's the `Note` component.
It uses `SpriteMaterial` and a `Sprite` like we covered in
[the article on billboards](threejs-billboards.html) 

```js
class Note extends Component {
  constructor(gameObject) {
    super(gameObject);
    const {transform} = gameObject;
    const noteMaterial = new THREE.SpriteMaterial({
      color: new THREE.Color().setHSL(rand(1), 1, 0.5),
      map: noteTexture,
      side: THREE.DoubleSide,
      transparent: true,
    });
    const note = new THREE.Sprite(noteMaterial);
    note.scale.setScalar(3);
    transform.add(note);
    this.runner = new CoroutineRunner();
    const direction = new THREE.Vector3(rand(-0.2, 0.2), 1, rand(-0.2, 0.2));

    function* moveAndRemove() {
      for (let i = 0; i < 60; ++i) {
        transform.translateOnAxis(direction, globals.deltaTime * 10);
        noteMaterial.opacity = 1 - (i / 60);
        yield;
      }
      transform.parent.remove(transform);
      gameObjectManager.removeGameObject(gameObject);
    }

    this.runner.add(moveAndRemove());
  }
  update() {
    this.runner.update();
  }
}
```

All it does is setup a `Sprite`, then pick a random velocity and move
the transform at that velocity for 60 frames while fading out the note
by setting the material's [`opacity`](Material.opacity). 
After the loop it the removes the transform
from the scene and the note itself from active gameobjects.

One last thing, let's add a few more animals

```js
function init() {

   ...

  const animalModelNames = [
    'pig',
    'cow',
    'llama',
    'pug',
    'sheep',
    'zebra',
    'horse',
  ];
+  const base = new THREE.Object3D();
+  const offset = new THREE.Object3D();
+  base.add(offset);
+
+  // position animals in a spiral.
+  const numAnimals = 28;
+  const arc = 10;
+  const b = 10 / (2 * Math.PI);
+  let r = 10;
+  let phi = r / b;
+  for (let i = 0; i < numAnimals; ++i) {
+    const name = animalModelNames[rand(animalModelNames.length) | 0];
    const gameObject = gameObjectManager.createGameObject(scene, name);
    gameObject.addComponent(Animal, models[name]);
+    base.rotation.y = phi;
+    offset.position.x = r;
+    offset.updateWorldMatrix(true, false);
+    offset.getWorldPosition(gameObject.transform.position);
+    phi += arc / r;
+    r = b * phi;
  }
```

{{{example url="../threejs-game-conga-line-w-notes.html"}}}

You might be asking, why not use `setTimeout`? The problem with `setTimeout`
is it's not related to the game clock. For example above we made the maximum
amount of time allowed to elapse between frames to be 1/20th of a second.
Our coroutine system will respect that limit but `setTimeout` would not.

Of course we could have made a simple timer ourselves

```js
class Player ... {
  update() {
    this.noteTimer -= globals.deltaTime;
    if (this.noteTimer <= 0) {
      // reset timer
      this.noteTimer = rand(0.5, 1);
      // create a gameobject with a note component
    }
  }
```

And for this particular case that might have been better but as you add
more and things you'll get more and more variables added to your classes
where as with coroutines you can often just *fire and forget*.

Given our animal's simple states we could also have implemented them
with a coroutine in the form of

```js
// pseudo code!
function* animalCoroutine() {
   setAnimation('Idle');
   while(playerIsTooFar()) {
     yield;
   }
   const target = endOfLine;
   setAnimation('Jump');
   while(targetIsTooFar()) {
     aimAt(target);
     yield;
   }
   setAnimation('Walk')
   while(notAtOldestPositionOfTarget()) {
     addHistory();
     aimAt(target);
     yield;
   }
   for(;;) {
     addHistory();
     const pos = history.unshift();
     transform.position.copy(pos);
     aimAt(history[0]);
     yield;
   }
}
```

This would have worked but of course as soon as our states were not so linear
we'd have had to switch to a `FiniteStateMachine`.

It also wasn't clear to me if coroutines should run independently of their
components. We could have made a global `CoroutineRunner` and put all
coroutines on it. That would make cleaning them up harder. As it is now
if the gameobject is removed all of its components are removed and
therefore the coroutine runners created are no longer called and it will
all get garbage collected. If we had global runner then it would be
the responsibility of each component to remove any coroutines it added
or else some other mechanism of registering coroutines with a particular
component or gameobject would be needed so that removing one removes the
others.

There are lots more issues a
normal game engine would deal with. As it is there is no order to how
gameobjects or their components are run. They are just run in the order added.
Many game systems add a priority so the order can be set or changed.

Another issue we ran into is the `Note` removing its gameobject's transform from the scene.
That seems like something that should happen in `GameObject` since it was `GameObject`
that added the transform in the first place. Maybe `GameObject` should have
a `dispose` method that is called by `GameObjectManager.removeGameObject`?

Yet another is how we're manually calling `gameObjectManager.update` and `inputManager.update`.
Maybe there should be a `SystemManager` which these global services can add themselves
and each service will have its `update` function called. In this way if we added a new
service like `CollisionManager` we could just add it to the system manager and not
have to edit the render loop.

I'll leave those kinds of issues up to you.
I hope this article has given you some ideas for your own game engine.

Maybe I should promote a game jam. If you click the *jsfiddle* or *codepen* buttons
above the last example they'll open in those sites ready to edit. Add some features,
Change the game to a pug leading a bunch of knights. Use the knight's rolling animation
as a bowling ball and make an animal bowling game. Make an animal relay race.
If you make a cool game post a link in the comments below.

<div class="footnotes">
[<a id="parented">1</a>]: technically it would still work if none of the parents have any translation, rotation, or scale <a href="#parented-backref">§</a>.
</div>