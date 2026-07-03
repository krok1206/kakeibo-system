// Vercel Serverless Function
// レシート画像をClaude APIに送信し、店名・日付・品目・金額をJSONで抽出する
//
// 事前準備: Vercelのプロジェクト設定 > Environment Variables に
//   ANTHROPIC_API_KEY = sk-ant-...
// を追加してください。(https://console.anthropic.com で取得)

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    res.status(405).json({ error: 'POSTメソッドのみ対応しています' });
    return;
  }

  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) {
    res.status(500).json({ error: 'サーバーにANTHROPIC_API_KEYが設定されていません' });
    return;
  }

  const { image, mediaType } = req.body || {};
  if (!image) {
    res.status(400).json({ error: '画像データがありません' });
    return;
  }

  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
  const finalMediaType = allowedTypes.includes(mediaType) ? mediaType : 'image/jpeg';

  const prompt = `これは日本のレシートの画像です。以下のJSON形式のみで出力してください。説明文やMarkdownのコードブロックは一切不要です。

{
  "store": "店名(不明なら空文字)",
  "date": "YYYY-MM-DD形式の日付(不明なら空文字)",
  "items": [
    { "name": "品目名", "amount": 数値(税込金額), "category": "食費/日用品/交通費/住居費/光熱費/通信費/娯楽費/医療費/被服費/交際費/その他 のいずれか" }
  ]
}

注意点:
- items は、レシートに複数の品目がある場合、可能な範囲で個別に分けてください。品目が多すぎる、または読み取りが難しい場合は、合計金額のみの1件にまとめても構いません。
- category は品目の内容から最も適切なものを推測してください。
- amount は数値のみ(カンマや円記号は含めない)。
- 割引・値引きはその品目の金額に反映するか、別途マイナスの品目として扱ってください。
- 読み取れない項目は空文字または0としてください。`;

  try {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': apiKey,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify({
        model: 'claude-sonnet-4-6',
        max_tokens: 1500,
        messages: [
          {
            role: 'user',
            content: [
              {
                type: 'image',
                source: { type: 'base64', media_type: finalMediaType, data: image },
              },
              { type: 'text', text: prompt },
            ],
          },
        ],
      }),
    });

    if (!response.ok) {
      const errBody = await response.text();
      res.status(502).json({ error: 'Claude APIの呼び出しに失敗しました', detail: errBody });
      return;
    }

    const data = await response.json();
    const textBlock = (data.content || []).find((b) => b.type === 'text');
    let raw = textBlock ? textBlock.text : '{}';

    // 念のためMarkdownのコードブロック記号を除去
    raw = raw.replace(/```json/g, '').replace(/```/g, '').trim();

    let parsed;
    try {
      parsed = JSON.parse(raw);
    } catch (e) {
      res.status(502).json({ error: 'レシートの解析結果を読み取れませんでした', raw });
      return;
    }

    res.status(200).json({
      store: parsed.store || '',
      date: parsed.date || '',
      items: Array.isArray(parsed.items) ? parsed.items : [],
    });
  } catch (err) {
    res.status(500).json({ error: 'サーバーエラーが発生しました', detail: String(err) });
  }
}
