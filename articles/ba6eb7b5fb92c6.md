---
title: "Next.jsでImageコンポーネントを使うと画像の下に謎の余白ができる問題の解決法"
emoji: "🏞️"
type: "tech"
topics: ["nextjs"]
published: true
---

Next.jsのImageコンポーネントをそのまま使うとなぜか画像の下に謎の余白が生成されます。(2021/10/13現在)
https://github.com/vercel/next.js/issues/18911#issuecomment-723362202
ここに解決法があるのでそのまま同じようにやります。

```jsx:修正前
<Image
    src={`/test.jpg`}
    width={360}
    height={240}
/>
```

```jsx:修正後
<>
    <div>
            <Image
                src={`/test.jpg`}
                width={360}
                height={240}
                />
    </div>
    <style jsx>
        {`
            .wrapper {
                letter-spacing: 0;
                word-spacing: 0;
                font-size: 0;
            }
        `}
    </style>
</>
```
要はImageの親コンポーネントのフォントサイズとかを0にしてあげることで治ります。