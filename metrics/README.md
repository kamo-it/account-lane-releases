# 利用規模の記録

`download-counts.csv` に、各リリースの `latest.yml` / `latest-mac.yml` の**累積ダウンロード数**を
毎日 00:00 JST に追記している（`.github/workflows/record-download-counts.yml`）。

## なぜこれで利用者数の見当がつくのか

electron-updater は更新チェックのたびに、**最新リリースの** `latest.yml`（Windows）/
`latest-mac.yml`（macOS）を取りに来る。この2ファイルは数百バイトのマニフェストで、
人が手動でダウンロードするものではない。つまり増分はほぼアプリからのアクセスであり、
**「更新チェックをした端末の延べ数」**の近似になる。

アプリに計測コードを入れずに済むのが利点。掲載しているプライバシー方針
（利用者のデータを当社サーバーへ送らない）と矛盾しない。

## 読み方

累積値なので、**日次の増分は自分で差分を取る**。

```bash
python3 - <<'PY'
import csv, collections
rows = list(csv.DictReader(open('metrics/download-counts.csv')))
series = collections.defaultdict(dict)
for r in rows:
    series[(r['tag'], r['asset'])][r['recorded_at']] = int(r['download_count'])

daily = collections.defaultdict(int)
for key, points in series.items():
    stamps = sorted(points)
    for prev, cur in zip(stamps, stamps[1:]):
        daily[cur[:10]] += points[cur] - points[prev]

for day in sorted(daily):
    print(day, daily[day])
PY
```

プラットフォーム別に見たいときは `asset` で絞る（`latest.yml` = Windows、`latest-mac.yml` = macOS）。

## 限界（ここを誤解しないこと）

- **ユニークではない。** 同じ端末が1日に複数回チェックすれば、その回数だけ数えられる。
  「1日あたりの延べ更新チェック数」であって DAU ではない。
- **起動していない端末は数えない。** 逆に、起動しっぱなしの端末は定期チェックで複数回数える。
- **新リリースを出すと、カウント先が新しいリリースの yml に移る。** 古いリリースの数値は
  そこで止まる。日次の合計を見るときは全リリースの増分を足すこと（上のスクリプトはそうしている）。
- **人手のアクセスも混ざる。** 疎通確認で `curl` すると1増える。リリース直後の数件は
  たいていそれ。
- **正確な DAU が必要になったら**、端末ごとの乱数IDのハッシュと日付だけを送る軽量な
  ping（Cloudflare Workers など）を別途検討する。個人情報や利用内容は送らない設計にすること。

## 実績の目安

v1.1.2 が最新だった約3日間で、`latest-mac.yml` が 18、`latest.yml` が 5。
母数が小さいうちは日次の揺れが大きいので、週単位の傾向で見るほうがよい。
