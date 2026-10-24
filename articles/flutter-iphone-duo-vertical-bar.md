---
title: "Flutter アプリを iPhone Duo の「縦帯」に対応させる"
emoji: "📱"
type: "tech"
topics: ["flutter", "ios", "iphoneduo", "dart", "xcode"]
published: true
---

こんにちは、ゆうです。

FlutterKaigi 2026のアプリの開発をしています。

Xを見ると、iPhone Duoを買った人が思ったより多いようでした。mixi2さんがFlutterでiPhone Duoに最適化しているポストも流れてきたので、FlutterKaigi 2026のアプリも対応することにしました。

https://x.com/chehonz201/status/2113578442751185351

以下の2つPRで対応してます。

https://github.com/FlutterKaigi/2026/pull/131

https://github.com/FlutterKaigi/2026/pull/132

後半に書いてある `verticalBarEdge` のガードは、後者の PR で入れています。

確認には Flutter 3.47.8、Xcode 27.1、iPhone Duo シミュレータ（iOS 27.1）を使っています。 ~~iPhone Duoを買うところに至っていないため、実機では確認していません。~~

## Xcode 27.1 でビルドすると、本文が縦帯の下に潜る

iOS 26 SDK（Xcode 26）でビルドしたアプリは、iPhone Duo では互換表示になります。全画面で表示するには、iOS 27.1 SDK（Xcode 27.1）でビルドし直す必要があります。

ビルドし直すと全画面にはなりますが、今度は「縦帯（vertical bar）」の問題が出ます。閉じた状態と、開いて横向きにしたとき、iOS はステータスバー（カメラ穴、時刻、Wi-Fi）を画面の片側に縦に並べます。ネイティブアプリなら、タブバーやツールバーもこの縦帯に移ります。

Flutter はタブバーを自前で描いているので、縦帯には移りません。`ListView` もスクロール方向（縦）の safe area しか避けないので、カードの右端が縦帯の下に潜りました。

![](/images/flutter-iphone-duo-vertical-bar/before-closed.png =300x)
*対応前。閉じた状態*

![](/images/flutter-iphone-duo-vertical-bar/before-open.png)
*対応前。開いて横向き*

シミュレータで測った safe area は次のとおりです（単位は pt）。

| 状態 | 画面サイズ | safe area（上・左・右・下） |
| --- | --- | --- |
| 閉じた縦向き | 466 × 678 | 0・0・84・34 |
| 閉じた横向き | 678 × 466 | 0・84・0・34 か 0・0・84・34（回す向きで入れ替わる） |
| 開いた横向き | 951 × 669 | 0・0・84・34 |
| 開いた縦向き | 669 × 951 | 上にステータスバーがあり、縦帯は出ない |

縦帯の幅は 84pt で、Flutter からは「上の inset が 0 で、左右の片側だけに 84pt の inset がある」ように見えます。

## タブを縦帯に入れる

 縦帯の上のほうはシステムの表示に任せ、その下にタブを縦に並べて下に寄せます。下部のタブバーはなくし、本文は縦帯の手前で止めて、背景色だけを画面の端まで続けます。

縦帯があるかどうかは、safe area の形で判定しています。開いた縦向きのように縦帯が消える状態や、閉じた横向きのように左右が入れ替わる状態があるので、機種では決めていません。

```dart:lib/core/util/vertical_bar_edge.dart
enum VerticalBarEdge {
  left,
  right;

  static VerticalBarEdge? fromPadding(EdgeInsets padding) {
    if (kIsWeb || defaultTargetPlatform != TargetPlatform.iOS || padding.top > 0) {
      return null;
    }
    return switch ((padding.left > 0, padding.right > 0)) {
      (true, false) => VerticalBarEdge.left,
      (false, true) => VerticalBarEdge.right,
      _ => null,
    };
  }
}
```

従来の iPhone は、縦向きなら上に 62pt、横向きなら左右の両方に 62pt の inset があるので、この条件には当たりません。iPad も上に inset があります。

あとは、go_router の `StatefulShellRoute` で作っているタブのシェルで、縦帯があるときだけレイアウトを切り替えます。縦帯には `NavigationRail` をそのまま置いています。

```dart:lib/core/ui/root_scaffold.dart
Widget _buildWithVerticalBar(BuildContext context, VerticalBarEdge edge) {
  final padding = MediaQuery.paddingOf(context);
  final onRight = edge == VerticalBarEdge.right;
  final barWidth = onRight ? padding.right : padding.left;

  final body = Expanded(
    child: MediaQuery.removePadding(
      context: context,
      removeLeft: !onRight,
      removeRight: onRight,
      child: Scaffold(body: navigationShell),
    ),
  );
  final bar = SizedBox(
    width: barWidth,
    child: _VerticalBarTabs(
      edge: edge,
      rail: NavigationRail(
        minWidth: barWidth,
        backgroundColor: Colors.transparent,
        groupAlignment: 1, // 下に寄せる
        labelType: NavigationRailLabelType.all,
        scrollable: true,
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: _onSelected,
        destinations: _railDestinations,
      ),
    ),
  );

  return Material(
    color: Theme.of(context).scaffoldBackgroundColor,
    child: Row(children: onRight ? [body, bar] : [bar, body]),
  );
}
```

本文側は `Row` で縦帯の分を空けているので、`MediaQuery.removePadding` で縦帯側の inset を消しています。消さないと、本文の `AppBar` がもう一度 84pt を避けます。

外側は `Scaffold` ではなく `Material` にして、`Scaffold` は本文側の1つだけにしています。`ScaffoldMessenger` は、登録された `Scaffold` のうち一番外側のものに SnackBar を出します。外側も `Scaffold` にすると、SnackBar が縦帯を含む全幅に出て、縦帯の一番下のタブを覆いました。`MaterialApp.builder` から出しているプッシュ通知の SnackBar も同じです。`Scaffold` を本文側だけにすれば、どこから出した SnackBar も本文の幅に収まります。

その代わり、キーボードを出しても縦帯は縮まないので、タブはキーボードの下に隠れます。iOS のタブバーもキーボードの下に隠れるので、それに合わせました。

「会場マップ」「アカウント」のような5文字のラベルも 84pt に収まりました。

![](/images/flutter-iphone-duo-vertical-bar/after-closed.png =300x)
*対応後。閉じた状態*

タブのシェルの外にある画面（設定画面や、ディープリンクで開く画面）は、縦帯の側だけ `SafeArea` を効かせるウィジェットで包んでいます。`Navigator.of(context, rootNavigator: true)` で push する画面もシェルの外に出るので、同じように包みます。

## カメラ穴

最初は、縦帯の上から 160pt を空けてタブを並べていました。縦向きのときのカメラ穴、時刻、Wi-Fi がだいたいその高さに収まっていたからです。

ところが、閉じたまま横向きにすると画面の高さが 466pt しかなく、5つ目のタブが入りきりませんでした。横向きではカメラ穴だけが出て、時刻と Wi-Fi は出ていなかったので、空ける量をカメラ穴の分（72pt）に減らしました。縦向きでも、タブは下寄せなので時刻や Wi-Fi とは重なりません。

その後、反対向きに回すと、カメラ穴が縦帯の**下端**に来ました。

![](/images/flutter-iphone-duo-vertical-bar/camera-overlap.png =500x)
*カメラ穴が「アカウント」のタブに重なっている*

このときの safe area は「右 84・下 34」で、カメラ穴が上にある縦向きと同じ値です。Flutter に届く情報だけでは、カメラ穴が上にあるのか下にあるのか分かりません。

iOS 27.1 では、カメラ穴などの位置を `UIView.reservedRegions(kind: .occlusion)` で取れます。これを AppDelegate から MethodChannel で返すようにしました。

```swift:ios/Runner/AppDelegate.swift
channel.setMethodCallHandler { [weak registrar] call, result in
  guard call.method == "occlusions" else {
    result(FlutterMethodNotImplemented)
    return
  }
  guard #available(iOS 27.1, *), let view = registrar?.viewController?.view else {
    result(nil) // 取れないときは nil にして、「カメラ穴がない」と区別する
    return
  }
  result(
    view.reservedRegions(kind: .occlusion).map { region in
      [
        "x": region.frame.minX, "y": region.frame.minY,
        "width": region.frame.width, "height": region.frame.height,
        "active": region.isActive,
      ]
    }
  )
}
```

`registrar.viewController?.view` は Flutter の描画面と同じ座標系なので、値はそのまま論理ピクセルとして使えます。閉じた縦向きでは (382, 0, 84, 174) が返ってきました。カメラ穴だけでなく、時刻と Wi-Fi の一帯まで含んだ範囲です。カメラ穴が下に来る向きでは (594, 384, 84, 82) と、カメラ本体の (611.7, 399.7, 37, 37) の2つでした。

Dart 側では、縦帯に重なる領域のうち画面の上半分にあるものを上の余白に、下半分にあるものを下の余白にして、その間にタブを並べます。

```dart:lib/core/ui/root_scaffold.dart
var top = _verticalBarCameraExtent; // 値を取れないときは上を 72pt 空ける
var bottom = padding.bottom;
final occlusions = _occlusions;
if (occlusions != null) {
  top = 0;
  final barRect = widget.edge == VerticalBarEdge.right
      ? Rect.fromLTRB(size.width - padding.right, 0, size.width, size.height)
      : Rect.fromLTRB(0, 0, padding.left, size.height);
  for (final occlusion in occlusions.where(barRect.overlaps)) {
    if (occlusion.center.dy < size.height / 2) {
      top = math.max(top, occlusion.bottom);
    } else {
      bottom = math.max(bottom, size.height - occlusion.top);
    }
  }
}
```

![](/images/flutter-iphone-duo-vertical-bar/camera-bottom-fixed.png =500x)
*カメラ穴が下にある向き*

![](/images/flutter-iphone-duo-vertical-bar/camera-top-left.png =500x)
*反対の向き。縦帯は左に移る*

この領域が変わったことを UIKit から知る方法は、見つけられませんでした。iOS 27.1 SDK のヘッダーにも、通知や delegate はありません。なので、最初の表示時、`didChangeMetrics`、アプリに戻ってきたとき（`AppLifecycleState.resumed`）に取り直しています。キーボードのアニメーション中は `didChangeMetrics` が毎フレーム呼ばれるので、前回と同じ結果なら再描画しないようにしています。

:::message alert
`reservedRegions` は iOS 27.1 SDK にしかないので、iOS アプリのビルドに Xcode 27.1 以上が必要になります。`#available(iOS 27.1, *)` は実行時のチェックなので、古い Xcode ではコンパイルが通りません。
:::

## `verticalBarEdge` をガードに使う

safe area から判定する形で一通り作ったあと、UIKit に `UITraitCollection.verticalBarEdge`（iOS 27.1）があるのを見つけました。システムが縦帯を置く辺を `Unspecified`、`Leading`、`Trailing` で返し、縦帯を使わない端末や状態では `Unspecified` になります。`systemTraitsAffectingVerticalBarEdge` を `registerForTraitChanges` に渡せば、変化も受け取れます。

これで判定を置き換えることも考えましたが、ガードとして足すだけにしました。縦帯の幅は結局 safe area から取る必要があります。それに、チャネル経由で非同期に届くので、開閉や回転の直後に safe area と食い違うおそれがあります。

辺の値も、そのままでは使えませんでした。開いた横向きで Split View にし（ホーム画面でアイコンを長押しして「Split View で開く」を選びます）、FlutterKaigi のアプリを左側に置くと、縦帯に接していないのに `Leading` が返ってきます。このとき safe area の左右は 0 なので、safe area で判定すれば縦帯なしになり、下部のタブバーが出ます。右側に置いたときは `Trailing` が返り、safe area の右に 84pt が入るので、縦帯にタブが並びます。

AppDelegate から値と変化を送り、`MaterialApp.builder` に置いたウィジェットで受け取ります。UIKit が `Unspecified` と答えているときは、safe area の形にかかわらず縦帯として扱いません。

```dart:lib/core/util/vertical_bar_edge.dart
static VerticalBarEdge? of(BuildContext context) {
  if (SystemVerticalBarScope.usesVerticalBarOf(context) == false) {
    return null;
  }
  return fromPadding(MediaQuery.paddingOf(context));
}
```

iOS 27.1 未満、Web、Android、それと値がまだ届いていないあいだは、safe area の形だけで判定します。

iPhone Duo で試した範囲では、`Unspecified` が返ったのは開いた縦向きだけでした。ここは上にステータスバーがあるので、safe area の形でも縦帯なしと判定されます。

### 回転の途中でレイアウトが崩れる

ガードを入れたあと、開いた横向きから縦向きに回すと、回転の途中でレイアウトが一瞬崩れるようになりました。ログを取ると、UIKit が `Unspecified` を知らせてから画面サイズが変わるまで、130〜150ms 空いていました。

そのあいだ、横向きのサイズ（951 × 669）のまま縦帯なしのレイアウトで組み直していたので、左に展開した `NavigationRail` と、縦帯の下まで伸びた本文が出ていました。シミュレータの画面を `xcrun simctl io <device> recordVideo` で録画してコマ送りすると、0.1 秒ほど映っています。

![](/images/flutter-iphone-duo-vertical-bar/rotation-before.png =500x)
*直す前。回転の途中に、左に展開したレールが出ている*

縦帯ありから `Unspecified` に変わるときだけ、次に画面サイズが変わる（`didChangeMetrics` が呼ばれる）まで反映を待つようにしました。サイズが変わらない場合に備えて、500ms たったら反映します。待っているあいだに縦帯の辺がまた届いたら、待つのをやめます。

```dart:lib/core/util/system_vertical_bar.dart
@override
void didChangeMetrics() {
  if (_pendingRemoval != null) {
    _apply(false);
  }
}

void _update(String? edge) {
  final usesVerticalBar = edge == null ? null : edge != 'unspecified';
  if (usesVerticalBar == false && _usesVerticalBar == true) {
    _pendingRemoval ??= Timer(verticalBarRemovalTimeout, () => _apply(false));
    return;
  }
  _apply(usesVerticalBar);
}

void _apply(bool? usesVerticalBar) {
  _pendingRemoval?.cancel();
  _pendingRemoval = null;
  if (mounted && usesVerticalBar != _usesVerticalBar) {
    setState(() => _usesVerticalBar = usesVerticalBar);
  }
}
```

最初の答えと、縦帯ありに変わるときは、すぐに反映します。縦帯ありの答えは safe area の形での判定を止めないだけなので、画面サイズが変わる前に反映しても見た目は変わりません。

直したあとは、回転中は横向きの縦帯のレイアウトのまま回り、縦向きのサイズになってから縦向きのレイアウトに変わります。閉じた横向きから開いて縦向きになるときは、画面サイズが変わった直後に `Unspecified` が届き、組み直しは縦向きのレイアウトの1回だけでした。

![](/images/flutter-iphone-duo-vertical-bar/rotation-after.png =500x)
*直したあと。同じ角度のコマでも、タブは右の縦帯に入ったまま*

## テスト

`tester.view` に iPhone Duo と同じサイズと safe area を入れれば、ウィジェットテストで縦帯を再現できます。`defaultTargetPlatform` で iOS に限っているので、`TargetPlatformVariant.only(TargetPlatform.iOS)` を付けます。

```dart
void _setView(WidgetTester tester, Size size, {double left = 0, double top = 0, double right = 0, double bottom = 0}) {
  const ratio = 3.0;
  tester.view.devicePixelRatio = ratio;
  tester.view.physicalSize = size * ratio;
  tester.view.padding = FakeViewPadding(
    left: left * ratio,
    top: top * ratio,
    right: right * ratio,
    bottom: bottom * ratio,
  );
  addTearDown(tester.view.reset);
}

testWidgets('moves the tabs into a vertical bar on the right', (tester) async {
  _setView(tester, const Size(466, 678), right: 84, bottom: 34);
  await _pumpShell(tester);

  final rail = tester.getRect(find.byType(NavigationRail));
  expect(rail.left, 382);
  expect(rail.right, 466);
}, variant: TargetPlatformVariant.only(TargetPlatform.iOS));
```

カメラ穴の位置と `verticalBarEdge` は、`setMockMethodCallHandler` でチャネルの応答を差し替えて試しました。

`verticalBarEdge` の変化の通知は、`tester.binding.defaultBinaryMessenger.handlePlatformMessage` で流しています。

反映を待つ処理は、`tester.pump(verticalBarRemovalTimeout - const Duration(milliseconds: 1))` のように時間を進めて、500ms の前後で表示が切り替わることを確かめました。これらのテストは、反映を待つ処理を外すと落ちます。

PR #131 の時点で、iPhone Duo シミュレータに加えて、iPhone 17 Pro のシミュレータ、Web、Android エミュレータでも、表示が崩れていないことを確かめました。ガードを入れたあとは、iPhone Duo と iPhone 17 Pro のシミュレータで確かめています。iPhone 17 Pro（iOS 26.3）ではチャネルが nil を返すので、表示は変わりません。Web と Android では、ガードは何もしません。実機では試していません。

## 対応していないこと・確かめていないこと

折り目には対応していません。Flutter が iOS の折り目を `MediaQuery.displayFeatures` で受け取れないためです（[flutter/flutter#192515](https://github.com/flutter/flutter/issues/192515)）。

iPad のウィンドウ表示で、safe area が「上は 0 で片側だけ」の形になるかどうかは確かめていません。

ヘッダーの説明どおりなら、その場合も `verticalBarEdge` のガードで除外されます。

ただ、iOS 27.1 のシミュレータランタイムは iPhone Duo にしか対応しておらず、iPad のシミュレータは iOS 27.0 までなので、`verticalBarEdge` 自体を取れません。
