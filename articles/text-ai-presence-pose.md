---
title: "\"なめらかに動かす\"のをやめたら、AIキャラに\"いる\"感が出た"
emoji: "🪆"
type: "tech"
topics: ["javascript", "css", "frontend", "ai", "個人開発"]
published: true
published_at: "2026-07-03 07:30"
---

テキストAIチャットに、マスコットを一体置いている。このキャラは、自分の写真をGeminiでイラスト化したものだ。子どもの頃から「絵に動きをつけて、キャラが動いているように見せたい」という願いがずっとあって、その延長で作った。

最初に断っておくと、目指していたのは「なめらかな動き」ではなかった。狙いはずっと「**どれだけ簡単に、いるように見せられるか**」のほうにあった。CSSアニメーションでぬるっと動かす道も試したけれど、それは目的ではなく選択肢のひとつでしかない。簡単さを突き詰めていったら、**ポーズ違いの止め絵を差し替える**という手に行き着いた。これは負けの代替ではなく、自分で選んだ答えだ。そして結果として、なめらかさを追わなかったのに、急にそのキャラが「いる」ように見え始めた。この経緯を、実装を交えて書く。

## "なめらかさ" は目的じゃなかった

やってみたかった動きは、キャラがぬるっと歩いたり、伸びをしたり、ジャンプしたりすることだ。CSSの `@keyframes` で `transform: translateY()` や `scale()` を打てば動く。実際、書いた。歩く絵も用意した。

確かにぎこちなかった。一枚絵を `translate` や `rotate` で動かしても、絵そのものは変形しないから「板が滑っている」感じが消えない。なめらかに見せようとするほど、機械が一枚の画像を動かしている感じが前に出てくる。

ただ、ここでつまずいて落ち込んだ、という話ではない。むしろこのぎこちなさが、いい材料になった。「そうか、自分は滑らかさが欲しかったわけじゃない。**一番簡単に"いる"を出したい**だけだ」とはっきりしたからだ。なめらかさはコストが高いわりに、その目的には直結していない。

もうひとつ、歩く絵を作って画面内をうろうろさせてみたら、チャットで考えごとをしている横でキャラが移動し続けるのは、単純にうるさかった。これも「動かせばいい」わけじゃないと教えてくれた。「動かす」より「動かさない」ほうが正しい場面がある。

そこで、簡単さを軸に方針を決めた。プログラムでなめらかに動かすのではなく、**止め絵を差し替える**ことで動きを表現する。パラパラ漫画と同じ理屈で、いちばん安くて、いちばん手早い。

## なぜ今 "差し替え" が現実的な手なのか

正直に書くと、自分は絵が苦手だ。子どもの頃にキャラを動かしたかった頃は、32×32や64×64のドット絵を、塗り絵みたいに一マスずつ埋めていた。一枚描くだけで一苦労で、ポーズ違いを何枚も用意するなんて現実的じゃなかった。だから当時は「プログラムで一枚の絵を動かす」くらいしか道がなかった。

今は違う。GeminiやMidjourneyに頼めば、同じキャラのポーズ違いを16枚、ほとんど一瞬で生成してくれる。**素材を量産するコストが劇的に下がった**から、「絵を差し替えて動かす」が現実的な選択肢になった。これは懐古ではなく技術的な理由だ。差し替え方式が成立するのは、ポーズ違いのアセットが安く手に入る今だからこそ、という前提がある。

## 解決＝ポーズ違いの止め絵を差し替える

マスコットのアセットは、PNG 16ポーズ。`<img>` の `src` を差し替えるだけで「絵が変わる」。これが表現の核になっている。

```javascript
const SORA_POSES = {
  default: 'sora-default.png',
  think: 'sora-think.png',
  coffee: 'sora-coffee.png',
  nod: 'sora-nod.png',
  walk1: 'sora-walk1.png',
  walk2: 'sora-walk2.png',
  look_left: 'sora-look_left.png',
  look_right: 'sora-look_right.png',
  sleep: 'sora-sleep.png',
  stretch: 'sora-stretch.png',
  jump: 'sora-jump.png',
  sitting: 'sora-sitting.png',
  peek: 'sora-peek.png',
  pointing_up: 'sora-pointing_up.png',
  surprised: 'sora-surprised.png',
  wave: 'sora-wave.png',
};
```

「絵を変える」操作は、突き詰めると一行だ。`<img>` の `src` を差し替えるだけ。状態に応じてCSSのクラスを掃除してから差し替える、という小さな関数にしてある。

```javascript
function clearMascotClasses() {
  soraMascot.classList.remove(
    'idle','blinking','swaying','bounce','thinking',
    'jumping','stretching','sleeping'
  );
}

function mascotPose(pose, duration) {
  soraMascotImg.src = SORA_POSES[pose] || SORA_POSES.default;
  if (duration) {
    setTimeout(() => {
      soraMascotImg.src = SORA_POSES[currentIdlePose()] || SORA_POSES.default;
    }, duration);
  }
}
```

重要なのは、ここで**CSSアニメーションは主役にしていない**こと。実際に「生きている」CSSは三つだけに絞った。

- `mascotBreathe` — idle時。3秒ループで `scaleY` を 1→1.015 にするだけの、ごく控えめな呼吸。
- `mascotThink` — 考えている時。1.5秒ループで ±3度の小さな揺れ。
- `mascotBounce` — 返答が出たとき。0.6秒、`translateY` で軽く跳ねる。

```css
@keyframes mascotBreathe {
  0%, 100% { transform: scaleY(1) translateY(0); }
  50%      { transform: scaleY(1.015) translateY(-1px); }
}
.sora-mascot.idle img {
  animation: mascotBreathe 3s ease-in-out infinite;
}

@keyframes mascotBounce {
  0%, 100% { transform: translateY(0); }
  30%      { transform: translateY(-12px); }
  50%      { transform: translateY(-6px); }
  70%      { transform: translateY(-10px); }
}
.sora-mascot.bounce img {
  animation: mascotBounce 0.6s ease;
}
```

つまり全体は **「どのPNGを出すか（差し替え）」×「ごく控えめなCSS」の二層**でできている。動きの大半は止め絵の差し替えが担い、CSSは呼吸と小揺れと跳ねの「微かな揺れ」だけを足す。なめらかなアニメで全部やろうとしていた頃より、ずっと「いる」感が出た。

## 動きを "会話" に乗せる＝既存関数のモンキーパッチ

次に効いたのが、マスコットを会話のフローに同期させる仕組みだ。ここは少し技術的に面白い。

マスコット機能はあとから足したので、チャット本体のコードには手を入れたくなかった。そこで、送受信フローの既存関数を**モンキーパッチ**——元のコードに手を入れず、外から動きを割り込ませる手法——でラップして、そこにマスコットの反応を割り込ませた。チャット側は自分が監視されていることを知らない、という疎結合（互いに知らない設計）な作りになっている。

送信した瞬間（タイピング表示が出る `appendTyping`）をフックして、コーヒーポーズに差し替え＋「考えている」揺れを付ける。

```javascript
// Override appendTyping to trigger mascot think (coffee pose)
const _origAppendTyping = appendTyping;
appendTyping = function() {
  const div = _origAppendTyping();
  mascotBusy = true;
  stopIdleBehavior();
  clearMascotClasses();
  showMascot('coffee');
  soraMascot.classList.add('thinking');
  return div;
};
```

返答が完了した瞬間（送信ボタンが `stop` から戻る `setSendButton`）をフックして、「できたよ」の指差しポーズ＋跳ね。2秒後に待機巡回へ戻す。

```javascript
// Hook into response completion
const _origSetSendButton = setSendButton;
setSendButton = function(mode) {
  _origSetSendButton(mode);
  if (mode !== 'stop') {
    // 返答完了 - pointing_up「できたよ」→ idle
    mascotBusy = false;
    lastInteraction = Date.now();
    clearMascotClasses();
    soraMascotImg.src = SORA_POSES.pointing_up || SORA_POSES.nod;
    soraMascot.classList.add('bounce');
    setTimeout(() => {
      clearMascotClasses();
      soraMascotImg.src = SORA_POSES[currentIdlePose()] || SORA_POSES.default;
      soraMascot.classList.add('idle');
      startIdleBehavior();
    }, 2000);
  }
};
```

送信したらコーヒーを飲みながら考え、返答が出たら指を立てて「できたよ」と跳ねる。たったこれだけの差し替えなのに、キャラがこちらの会話に反応している感じが一気に強くなった。元のチャットロジックには一行も触っていない。

## 止まっている時間こそ "いる" 感

意外だったのは、何もしていない待機時間の作り込みが一番効いたことだ。

待機中は、ポーズを固定の順番で巡回させている。`setInterval` ではなく `setTimeout` の再帰連鎖にして、15秒ごとに次のポーズへ送る。

```javascript
const IDLE_POSE_CYCLE = ['sitting', 'peek', 'pointing_up', 'wave', 'sitting', 'peek'];
const IDLE_POSE_INTERVAL = 15000;

function scheduleNextIdle() {
  idleTimer = setTimeout(doIdleAction, IDLE_POSE_INTERVAL);
}

function doIdleAction() {
  if (mascotBusy || soraMascot.classList.contains('hidden')) return;
  // 次のポーズへ巡回
  idleCycleIndex = (idleCycleIndex + 1) % IDLE_POSE_CYCLE.length;
  clearMascotClasses();
  soraMascotImg.src = SORA_POSES[currentIdlePose()] || SORA_POSES.default;
  updateMascotFlip();
  soraMascot.classList.add('idle');
  scheduleNextIdle();
}
```

ここで一つ、あえての選択をしている。**ランダムにしなかった**。瞬きの間隔をランダムにすると機械っぽさが消える、とよく言われるし、自分も前はそう書いた。でも止め絵の巡回でランダムを入れると、かえって挙動が読めず落ち着かなかった。固定15秒で淡々と座る・覗く・指を立てる・手を振るを繰り返すほうが、生活のリズムみたいに見えて馴染んだ。

`peek`（壁から乗り出すように覗く）ポーズのときだけ、画面の左半分にいたら `scaleX(-1)` で左右反転させて、向きを内側に合わせている。

```javascript
function updateMascotFlip() {
  const pose = currentIdlePose();
  if (pose !== 'peek') { soraMascot.classList.remove('flip'); return; }
  const rect = soraMascot.getBoundingClientRect();
  const centerX = rect.left + rect.width / 2;
  if (centerX < window.innerWidth / 2) {
    soraMascot.classList.add('flip');
  } else {
    soraMascot.classList.remove('flip');
  }
}
```

```css
.sora-mascot.flip img { transform: scaleX(-1); }
```

体感として一番よく出るのは、この `peek` の覗き込みだ。仕事の片隅から、こちらを覗いている。歩かせるのをやめて引き算した結果、止まっているのにいちばん「いる」感が出たのが、この覗きだった。

## 正直な余談：繋いでいない動きの化石

ここまで読むと、もっと色々動きそうに見えるかもしれない。実際、コードには**繋いでいない動きの化石**が残っている。

CSSには `mascotBlink`（瞬き）、`mascotSleep`（放っておくと眠る）、`mascotSway`、`mascotJump`、`mascotStretch` の `@keyframes` がちゃんと書いてある。`SORA_POSES` にも `sleep`、`jump`、`stretch` のPNGがある。

```css
/* 書いたが、JSからクラスを付けていない＝発火しない */
@keyframes mascotBlink {
  0%, 85%, 100% { transform: scaleY(1); }
  90% { transform: scaleY(0.95); }
  92% { transform: scaleY(0.85); }
  95% { transform: scaleY(1); }
}
.sora-mascot.blinking img { animation: mascotBlink 4s ease-in-out; }

@keyframes mascotSleep {
  0%, 100% { transform: translateY(0) rotate(0deg); }
  50%      { transform: translateY(2px) rotate(-2deg); }
}
.sora-mascot.sleeping img { animation: mascotSleep 2.5s ease-in-out infinite; }
```

でも、JS側でこれらの `blinking` / `sleeping` クラスを付けている箇所はない。`clearMascotClasses()` が「消す対象」として名前を知っているだけで、誰も付けない。だから現状、瞬きも睡眠もジャンプも伸びも、画面では起きない。「止まっていても動いて見えるように瞬かせたい」「かまわないと眠らせたい」という当初の願いの、書きかけのまま残った化石だ。強調しておきたいのは、これは技術的に**できなかった**のではなく、目的に届いていたから**止めた**ということ。呼吸・小揺れ・跳ねの三つと止め絵の差し替えで「一番簡単に"いる"を出す」には十分で、これ以上配線する必要がなかった。化石が残っているのは、設計ミスというより、作りながら設計が育った跡だと思っている。消さずに残してあるのは、いつか気が向いたら繋ぐかもしれないから、くらいの理由でしかない。

## まとめ

滑らかに動かすことを目指していたわけじゃなかった。「**一番簡単に、いるように見せる**」を追っていったら、止め絵の差し替えに行き着いた。これが芯だ。

- 目的は"なめらかさ"ではなく"どれだけ簡単に動かせるか"。差し替えはその答え。
- ポーズ違いの**止め絵を差し替える**のは、安く速く、しかも「いる」感が出た。
- CSSは呼吸・小揺れ・跳ねの**微かな揺れだけ**に絞る。主役は差し替え。
- 会話フローには既存関数の**モンキーパッチ**で割り込めば、本体に触らず同期できる。
- 待機の巡回はあえて**ランダムにしない**。歩きは**引き算**した。止まっているほうが「いる」。

最後に本番向けの注意を一つ。`prefers-reduced-motion: reduce` を尊重して、呼吸や跳ねのCSSアニメーションを止める分岐は入れておきたい。動きで「いる」感を出す実装ほど、動きを止めたい人への配慮が要る。

子どもの頃に「少しだけ動かしてキャラが動いているように見せたい」と思っていたものは、なめらかなアニメではなく、止め絵の差し替えの先にあった。素材が一瞬で手に入る今だからこそ、その一番簡単な道が通れるようになった。
