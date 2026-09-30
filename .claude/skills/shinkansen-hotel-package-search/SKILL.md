---
name: shinkansen-hotel-package-search
description: 新幹線＋ホテルのセットプランを、実際の日付・人数・喫煙条件でEXダイナミックパック・日本旅行・JTBの3サイトで検索し、価格・アクセス・チェックイン/アウト・写真・リンク付きのHTML比較表にするときに使う。
---

# 新幹線＋ホテル セットプラン比較

新幹線とホテルのセットプランを、ユーザーの**実際の条件**で複数サイトから取得し、比較表として返す。予約の確定はユーザーが行う（このスキルでは予約しない）。

## 大原則

- **実際の条件で検索した料金以外は書かない。** 特集ページの「代金例」は別の日付・別の人数（例：2名1室）の数字なので、絶対に引用しない。
- 料金は変動型。**取得日時を必ず明記**する。
- 各ホテルに**Webサイトリンクを必ず付ける**。
- 出典（Sources）を最後に並べる。

## 1. 条件をそろえる

足りない項目だけ質問する（AskUserQuestion）。

- 往路日・泊数・復路日
- 出発駅・宿泊エリア（到着駅）
- 人数と部屋割り（1名1室／2名1室）。大人1名あたりの料金は部屋割りで変わる
- 喫煙／禁煙
- 予算の意味。特に指定がなければ「ホテル実質代＝セット代金−通常の往復運賃」として扱い、そう明記する
- 現地への到着予定時刻（仕事終わりなど）。チェックイン締切の判定に使う
- 会場や希望エリア（あれば）

## 2. 予算のしきい値を計算する

1. 通常期・のぞみ普通車指定席の往復運賃を検索して確認する（例：東京〜名古屋は往復22,600円。2026年9月時点）。
2. セット代金の上限 = ホテル予算 + 往復運賃。
3. 表には「ホテル実質＝セット代金−往復運賃（近似）」の考え方を一行で添える。

## 3. 検索するサイト

東海道・山陽新幹線沿線の場合、次の3つを**必ず全部**見る。同じホテルでも価格が逆転することがあるため。

1. EXダイナミックパック（EX旅パック）
2. 日本旅行「JR・新幹線＋ホテル」
3. JTB「新幹線・JR＋ホテル」

**検索結果はJavaScriptで描画されるため、WebFetchでは0件になる。** ブラウザ（ユーザー設定の優先ブラウザ。built-in-browser スキルを先に読む）で操作する。

### 3a. 日本旅行

検索結果のURL形式：

```
https://www.nta.co.jp/dp-jr/hotels/{宿泊地コード}?depAreaCode={出発地コード}&goDate=YYYYMMDD&nights={泊数}&adultPax={人数}&roomCount=1&kodawari=J1-smokingRoom&sort=lowPrice&page={n}
```

- 分かっているコード：出発地 `T31`＝首都圏、宿泊地 `223`＝名古屋市内。それ以外はUIで条件を設定し、URLから読み取る。
- 喫煙ルームの絞り込みは `kodawari=J1-smokingRoom`。禁煙など他の条件は「こだわり条件を追加する」で設定し、URLから読み取る。
- Cookieバナーは「同意しない」を選ぶ。
- 1ページ10件。セット代金の上限を超えるまでページをめくる。
- 絞り込みなしだと、一覧の最安プランが禁煙のことがある。**必ず喫煙／禁煙の条件で絞り込む。**
- ホテルIDは、`main a[href*="/dp-jr/hotels/"]` のhrefから取れる（例 `5450-312`）。
- 個別ホテルへのリンク：`https://www.nta.co.jp/dp-jr/hotels/{ID}/rooms?depAreaCode=T31&goDate=YYYYMMDD&nights=1&adultPax=1`
- 料金は「往復追加代金なし列車・普通車指定席」が前提。

### 3b. EXダイナミックパック

1. 検索フォームを開く：`https://travel.jr-central.co.jp/top/dpsitetop/`（JavaScriptのフォーム）
   - 方面・都道府県・エリア、往路の出発駅・到着駅：select要素なので `form_input` で設定する
   - 旅行期間：入力欄をクリック → ▶で月を送る → 出発日と帰着日をクリック → OK
   - 人数：入力欄をクリック → ＋／−で調整 → OK
   - 「さらに絞り込む」→「部屋のこだわり」を開く → 「喫煙ルーム」にチェック
   - 検索：テキストが「検索」のボタンをJavaScriptでclickする
2. 結果ページ（`/extraindp/extrainHotelList?...&kodawari=smoking_room_class&sort=lowPrice&page=N...`）は、`page=` を変えれば続きを取れる。
3. 駅コード：東京010／品川020／新横浜030／静岡080／浜松100／名古屋130／京都160／新大阪170／新神戸180／岡山220／広島280／博多350。エリアコードの例：名古屋2301。
4. ホテルIDは、カード要素のHTMLから正規表現 `/54\d\d[A-Z]\d\d/` で抜き出す。接頭辞は5445・5446など複数ある。
5. ホテル個別ページ（`/extraindp/planListByRooms?hotelId=...`）には、**列車を特定した `extrains` パラメータが必須**。汎用の値では400／500エラーになる。一度どれかのホテルの「プラン一覧」をクリックし、そのURLの `extrains` を控えて、同じ検索の他のホテルにも使い回す。
6. 既定の料金は特定の列車を前提にしている（例：行きのぞみ1号 6:00発、帰りのぞみ64号 22:12発）。その列車を表に注記する。

### 3c. JTB

1. 入口：`https://www.jtb.co.jp/kokunai_jr/`
2. 混雑時は「Web待合室」（`aux.jtb.co.jp/waiting/`）に回される。数時間待つこともある（実例：13:03に開いて、入れる目安は14:57）。
   - 待合室のタブは閉じない。閉じたり別のブラウザで開き直したりすると、順番がリセットされることがある。
   - 待てないときは、JTBの列を「未確認（混雑）」にして、EXと日本旅行だけで表を出す。そのことを表の前に書く。
3. 出発地・行き先・日付・泊数・人数・部屋数をUIで設定して検索し、結果ページのURLを控える。喫煙／禁煙の条件でも必ず絞り込む。
4. 一覧の料金がどの列車を前提にしているかを、列車選択の画面で確かめて表に注記する。
5. 乗車日の1か月より前は「リクエスト受付」になる。列車は第三希望までの申し込み、座席は条件の指定だけで、その場では確定しない。

## 4. チェックイン／チェックアウト時刻を取る

EXのプランページにある「In HH:MM～HH:MM / Out HH:MM」を使う。EXのページを開いた状態で、同じオリジンからfetchして一括で取得する。

```js
const T = '<planListByRoomsのURL。hotelId=ID の部分をプレースホルダにする>';
const ids = [/* ホテルID */];
const res = {};
await Promise.all(ids.map(async id => {
  const t = await (await fetch(T.replace('ID', id))).text();
  const d = new DOMParser().parseFromString(t, 'text/html');
  const txt = (d.querySelector('main') || d.body).textContent.replace(/\s+/g, ' ');
  const toks = txt.match(/In ?\d{1,2}:\d{2}(?: ?～ ?\d{1,2}:\d{2})? ?\/ ?Out ?\d{1,2}:\d{2}|\d{2},\d{3}円/g) || [];
  res[id] = d.title.split('の宿泊')[0] + ' :: ' + toks.slice(0, 8).join(' | ');
}));
```

- 並びは「時刻 → 料金」の順。先頭が喫煙ルームの最安プランに当たる。料金が一覧と一致するか照合する。
- 24時を超える表記は読み替えて書く（26:00＝翌2時）。
- 日本旅行やJTBにしかないホテルは、個別ページを開いて読み取る。

## 5. アクセスと写真を取る

日本旅行のページ（`https://www.nta.co.jp/` のどこか）を開いた状態で、同じオリジンからfetchして一括で取る。

- アクセス：「施設・アクセス」ページ（`/dp-jr/hotels/{ID}/detail`）の「アクセス情報」欄。
- 写真：「写真」ページ（`/dp-jr/hotels/{ID}/images`）のHTMLに埋め込まれたJSON。`typeCode` は 1＝外観、2＝館内、3＝客室、5＝朝食。外観・客室・館内（なければ朝食）の順に3枚選ぶ。

```js
const q = 'depAreaCode=T31&goDate=YYYYMMDD&nights=1&adultPax=1';
const ids = [/* 日本旅行のホテルID */];
const res = {};
await Promise.all(ids.map(async id => {
  const [dt, im] = await Promise.all(['detail', 'images'].map(p =>
    fetch(`/dp-jr/hotels/${id}/${p}?${q}`).then(r => r.text())));
  const d = new DOMParser().parseFromString(dt, 'text/html');
  const label = [...d.querySelectorAll('dt, th')].find(e => e.textContent.trim() === 'アクセス情報');
  const photos = [...im.matchAll(/"typeCode":"(\d+)","url":"([^"]+)","caption":"([^"]*)"/g)]
    .map(m => ({type: m[1], url: m[2], caption: m[3]}));
  const pick = [];
  for (const t of ['1', '3', '2', '5']) {
    // 客室はバス・トイレなどより、部屋全体が写った写真を優先する
    const p = photos.find(p => p.type === t && !/バス|トイレ|アメニティ/.test(p.caption)) || photos.find(p => p.type === t);
    if (p && pick.length < 3) pick.push(p);
  }
  for (const p of photos) if (pick.length < 3 && !pick.includes(p)) pick.push(p);
  res[id] = {
    name: d.title.split('の施設')[0],
    access: label?.nextElementSibling?.textContent.trim().replace(/\s+/g, ' '),
    photos: pick,
  };
}));
```

- アクセスは、サイトの記載を短くしてそのまま書く（例：JR名古屋駅より徒歩約4分）。
- アクセス欄が空のホテルや、日本旅行に載っていないホテルは、EX・JTBのホテルページかホテル公式サイトで確かめる。見つからなければ「要確認」と書く。
- 写真のURLは参照元なしでも表示できる（`nta.co.jp/gallery/images/...`）。そのままHTMLに貼ってよい。写真が取れないホテルは「写真なし」と書く。

## 6. 間違えやすい点（確認リスト）

- 特集ページの「代金例」は使わない（別の日付・2名1室の数字）。
- 金曜・土曜の宿泊は平日より高い。
- 1名1室と2名1室では、1名あたりの料金が違う。
- 早期予約プラン（「早期申込N日前」「早トクN」）は、**締切日＝出発日−N日**を計算して書く。
- 列車の前提が違う（EX：既定の列車、日本旅行：追加代金なし列車、JTB：列車選択の画面で確かめる）。実際に乗る列車で料金を確認するよう注記する。
- EXの予約にはEX会員（スマートEXは年会費無料）が必要。当日の列車変更は、改札前・発車前なら何度でもできる。
- 日本旅行の列車変更は、出発7日前まで・往復で1回。
- JTBは、予約後に列車・座席を変えられない。変えるには全部取り消して予約し直す（取消料は自分で負担）。取消料は出発日の20日前からかかる。
- JTBの予約は出発前日の23:49まで（一部のプランはもっと早く締め切る）。会員登録なしのゲスト予約もできる。
- 到着予定時刻がチェックイン締切より遅いホテルには ⚠ を付ける。

## 7. 出力形式（HTML）

結果はHTMLファイルにまとめる。写真を並べて見比べられるようにするため。

1. `~/Downloads/shinkansen-hotel-{往路日YYYYMMDD}-{宿泊エリア}.html` に書き出す（例：`shinkansen-hotel-20261030-nagoya.html`）。
2. `open <ファイルのパス>` でブラウザに開く。
3. チャットには、ファイルの場所と、条件に合う最安の1〜2件だけを短く書く。

ページの中身：

- 先頭に、条件（日付・人数・部屋・喫煙）、予算のしきい値、取得日時を一行で書く。
- 比較表を、3サイトのうち一番安い料金の順に並べる。列は次のとおり。

| ホテル（リンク＋写真3枚） | アクセス | EXダイナミックパック | 日本旅行 | JTB | 最安 | チェックイン | チェックアウト |
|---|---|---|---|---|---|---|---|

- 載っていないサイトは「掲載なし」、JTBに入れなかったときは「未確認（混雑）」と書く。
- 各行で一番安い料金を太字にする。「最安」列には、そのサイト名と2番目との差を書く（例：日本旅行（EXより3,430円安い））。
- 表の後に、列車の前提、早期予約の締切、会員登録の要否、列車変更の条件、チェックイン締切の注意を短く書く。
- 最後に Sources（検索結果ページのURL）を並べる。

ひな形（行は1ホテル分）：

```html
<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>新幹線＋ホテル比較 {往路日} {宿泊エリア}</title>
<style>
  body { font-family: system-ui, sans-serif; margin: 16px; line-height: 1.5; }
  .wrap { overflow-x: auto; }
  table { border-collapse: collapse; }
  th, td { border: 1px solid #ccc; padding: 6px 8px; vertical-align: top; }
  th { background: #f3f3f3; }
  .photos { display: flex; gap: 4px; margin-top: 6px; }
  .photos img { width: 120px; height: 90px; object-fit: cover; }
</style>
</head>
<body>
<h1>新幹線＋ホテル比較（{往路日}〜{復路日}・{宿泊エリア}）</h1>
<p>条件：… ／ 予算のしきい値：… ／ 取得日時：…</p>
<div class="wrap"><table>
<tr><th>ホテル</th><th>アクセス</th><th>EXダイナミックパック</th><th>日本旅行</th><th>JTB</th><th>最安</th><th>チェックイン</th><th>チェックアウト</th></tr>
<tr>
  <td><a href="{ホテルのリンク}">{ホテル名}</a>
    <div class="photos">
      <a href="{写真URL}"><img src="{写真URL}" alt="{caption}" title="{caption}" loading="lazy"></a>
      <!-- 写真はあと2枚、同じ形で並べる -->
    </div></td>
  <td>{アクセス}</td>
  <td>{EXの料金}</td><td><b>{日本旅行の料金}</b></td><td>{JTBの料金}</td><td>{最安のサイトと2番目との差}</td>
  <td>{チェックイン}</td><td>{チェックアウト}</td>
</tr>
</table></div>
<h2>注意</h2>
<ul><li>…</li></ul>
<h2>Sources</h2>
<ul><li><a href="…">…</a></li></ul>
</body>
</html>
```

## 8. やらないこと

- 予約の確定、ログイン、個人情報の入力。プランの「選択」を押した後の人数・性別の画面から先には進まない。
- 確認していない料金や時刻を書くこと。
