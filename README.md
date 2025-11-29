# Utility Functions 

## Table of content
1. 
2. 


Copy the code blocks into a `.js` file and run with `node file.js`.


# size

**Description**: Return the "size" of different data types:

-   Arrays and strings: `.length`
    
-   Objects: number of own enumerable properties
    
-   Numbers: number of digits in their string representation
    
-   Falsy values -> `0`
    

```js
const size = (data) => {
  if (data === null || data === undefined) return 0;
  if (Array.isArray(data)) return data.length;
  if (typeof data === "string") return data.length;
  if (typeof data === "object") return Object.keys(data).length;
  if (typeof data === "number") return String(data).length;
  return 0;
};

// Tests / Examples
console.log(size([1,2,3]));            // 3
console.log(size('hello'));            // 5
console.log(size({a:1, b:2}));         // 2
console.log(size(12345));              // 5
console.log(size(null));               // 0
console.log(size(undefined));          // 0
console.log(size(true));               // 0 // booleans are not handled

```

----------

## after

**Description**: Returns a function that will invoke `cb` only after it has been called `n` times. The `cb` is called once when the count reaches `n`.

```js
function after(n, cb) {
  let count = 0;
  return function (...args) {
    count++;
    if (count === n) {
      return cb.apply(this, args);
    }
  };
}

// Example: execute callback only after 3 calls
const sayHiAfter3 = after(3, () => console.log('Hi after 3 calls'));
sayHiAfter3(); // nothing
sayHiAfter3(); // nothing
sayHiAfter3(); // prints "Hi after 3 calls"

// Test: using return value
let called = false;
const r = after(2, (x) => { called = true; return x * 2; });
console.log(r(3)); // undefined (first call)
console.log(r(5)); // 10 (second call triggers cb)
console.log('called?', called); // true

```
**Example 2:** This helper demonstrates how `after` can be used to run a final callback when several asynchronous uploads finish.
```js
function uploadFile(file, callback) {
  console.log(`Uploading ${file}...`);

  setTimeout(() => {
    console.log(`${file} uploaded`);
    callback();
  }, Math.random() * 2000);
}

const files = ['abc.png', 'cdf.pdf', 'image.jpg'];
const afterAll = after(files.length, () => console.log('ALL FILES ARE UPLOADED.'));

files.forEach((file) => uploadFile(file, afterAll))
```

----------

## once

**Description**: Creates a function that calls `cb` at most once. Subsequent calls return `undefined` (or could return the cached value if you extend it).

```js
function once(cb) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = cb.apply(this, args);
      return result;
    }
    return result; // returning same result on subsequent calls is often useful
  };
}

// Example
function log() {
  console.log('I WILL PRINT ONCE');
  return 42;
}
const logPrint = once(log);
console.log(logPrint()); // prints and returns 42
console.log(logPrint()); // does not print, returns 42

// Test: ensure it only runs once
let runs = 0;
const runOnce = once(() => ++runs);
console.log(runOnce()); // 1
console.log(runOnce()); // 1
console.log('runs now:', runs); // 1

```
----------

## get

**Description**: Safely get deep properties from an object using a dot-and-bracket path string (or array of segments). Returns a default value when the path is not present.

```js
function get(obj, path, defaultValue) {
  if (obj === null || obj === undefined || !path) return defaultValue;

  const segments = Array.isArray(path)
    ? path
    : path
        .replace(/\[(\d+)\]/g, '.$1') // convert [3] -> .3
        .split('.')
        .filter(Boolean);

  let result = obj;

  for (let key of segments) {
    if (result === null || result === undefined) return defaultValue;
    if (typeof result === 'object' && !Object.prototype.hasOwnProperty.call(result, key)) {
      return defaultValue;
    }
    result = result[key];
  }

  return result === undefined ? defaultValue : result;
}

// Examples / Tests
const data = { a: { b: [{ c: 3 }] } };
console.log(get(data, 'a.b[0].c', 'x')); // 3
console.log(get(data, ['a','b','0','c'], 'x')); // 3
console.log(get(data, 'a.b[1].c', 'not found')); // 'not found'
console.log(get(null, 'a.b', 'nope')); // 'nope'

```

----------

## memoize

**Description**: Cache results of `fn` by a key. If `resolver` is provided, use it to generate the cache key. Handles Promise results so concurrent calls can share an in-flight promise and the cache stores the resolved value later.

```js
function memoize(fn, resolver) {
  const cache = new Map();

  return function (...args) {
    const now = typeof performance !== 'undefined' ? performance.now() : Date.now();

    const key = resolver ? resolver(...args) : JSON.stringify(args);

    if (cache.has(key)) {
      // cached either resolved value or a Promise
      return cache.get(key);
    }

    const result = fn(...args);

    if (result instanceof Promise) {
      // store promise so concurrent callers share it
      const p = result.then((value) => {
        cache.set(key, value);
        return value;
      });
      cache.set(key, p);
      return p;
    }

    cache.set(key, result);
    return result;
  };
}

// Example: synchronous
const slowAdd = (a, b) => {
  // simulate heavy compute
  for (let i = 0; i < 1e6; i++); // nop
  return a + b;
};
const memoAdd = memoize(slowAdd);
console.time('first'); console.log(memoAdd(2,3)); console.timeEnd('first');
console.time('second'); console.log(memoAdd(2,3)); console.timeEnd('second');

// Example: async (fetch-like)
const fetchData = memoize(
  async (endpoint, params) => {
    const url = `${endpoint}?${new URLSearchParams(params).toString()}`;
    const res = await fetch(url);
    return res.json();
  },
  (endpoint, params) => `${endpoint}-${JSON.stringify(params)}`
);

// Note: the fetch example requires a runtime with fetch (node 18+ or browser)

```

**Note**: -   When memoizing async functions, be careful with error handling — if the promise rejects you may want to remove the cache entry so subsequent calls can retry

----------

## isEqual (React usecase)
  

React's `useEffect` performs **shallow comparison** on dependencies. This means that:
- Objects, arrays, and functions are considered “changed” even if their contents did not change.
- Any recreated object triggers the effect again.
### Problem 
```js
setUser({ ...user })
```

Even though the data is the same, the reference is new — causing `useEffect` to run again.

### Solution  
`useDeepCompareEffect` performs a **deep comparison** of dependencies.

It only triggers the effect when the **actual content** changes, not the reference.
Useful when:
- Dependencies are nested objects
- You want to avoid unnecessary re-renders
- State updates recreate objects frequently
---

### `useDeepCompareEffect` Hook

  

```js
import { useEffect, useRef } from  "react";

export  function  useDeepCompareEffect(effect:  any, deps:  any) {
const prevDeps =  useRef(null);

if (!isDeepEqual(prevDeps.current, deps)) {
     prevDeps.current = deps;
}

useEffect(effect, [prevDeps.current]);

}

  
export  function  isDeepEqual(a:  any, b:  any) {
    if (a === b) return  true;
	if (
	typeof a !==  "object"  ||
	typeof b !==  "object"  ||
	a ===  null  ||
	b ===  null
	) return  false;

const aKeys = Object.keys(a);
const bKeys = Object.keys(b);

if (aKeys.length !== bKeys.length) return  false;
 
for (let key of aKeys) {
 if (!isDeepEqual(a[key], b[key])) return  false;
}

return  true;
}

```


---

### `Profile` component for solution

```js
import { useEffect, useState } from "react";
import { useDeepCompareEffect } from "./hooks/useDeepCompareEffect";

export default function Profile() {
  const [user, setUser] = useState({
    name: "Alice",
    address: { city: "NYC", zip: 10001 },
  });

  // Shallow comparison
  useEffect(() => {
    console.log("useEffect detected change in user:", user);
  }, [user]);

  // Deep comparison (custom hook)
  useDeepCompareEffect(() => {
    console.log("useDeepCompareEffect detected deep change in user:", user);
  }, [user]);

  return (
    <div className="profile-container">
      <div className="profile-json">
        {JSON.stringify(user, null, 2)}
      </div>

      <div className="profile-actions">
        <button
          className="profile-button"
          onClick={() =>
            setUser((prev) => ({
              ...prev,
              address: { ...prev.address, city: "Boston" },
            }))
          }
        >
          Change City to Boston
        </button>

        <button
          className="profile-button secondary"
          onClick={() => setUser((prev) => ({ ...prev }))}
        >
          Shallow Rerender (no content changes)
        </button>
      </div>
    </div>
  );
}
```

## compose

**Description**: Compose functions right-to-left. The last function in `fns` will receive the full argument list; subsequent functions receive the previous function's single result.

```js
const compose =
  (...fns) =>
  (...args) => {
    return fns.reduceRight((acc, fn, index) => {
      return fns.length - 1 === index ? fn(...acc) : fn(acc);
    }, args);
  };

// Small helpers
const add = (a, b) => a + b;
const double = (x) => x * 2;
const square = (x) => x * x;
const addby10 = (x) => x + 10;

const calculate = compose(addby10, square, double, add);
console.log(calculate(5, 5)); // add(5,5)=10 -> double(10)=20 -> square(20)=400 -> addby10(400)= 410

// Simple test
console.log(calculate(1,2)); // add(1,2)=3 -> double(3)=6 -> square(6)=36 -> +10 => 46

```
----------

## deepPick


**deepPick** is a smart way to choose the most important parts of your data or model so everything runs faster, lighter, and more efficiently. Instead of using *all* data, deepPick focuses on the pieces that actually matter.

## 🌟 What it does
- Picks the most useful data samples  
- Removes noise and unnecessary information  
- Helps models train faster  
- Saves memory and compute power  

```js

function deepPick(obj, paths) {
  const result = {};

  for (const path of paths) {
    const keys = path.split(".");
    const value = getValue(obj, keys);
    if (value !== undefined) assign(result, keys, value);
  }

  return result;
}

function getValue(obj, keys) {
  if (!obj) return undefined;
  const [first, ...rest] = keys;
  if (!rest.length) return obj[first];
  return getValue(obj[first], rest);
}

function assign(target, keys, value) {
  const [first, ...rest] = keys;
  if (!rest.length) {
    target[first] = value;
    return;
  }
  if (!target[first]) target[first] = {};
  assign(target[first], rest, value);
}


const apiResponse = {
  user: {
    id: 42,
    name: { first: "Alicia", last: "Wells" },
    email: "alicia@example.com",
    address: {
      city: "Austin",
      zip: "73301"
    }
  },
  meta: { timestamp: 1732551820 }
};

const picked = deepPick(apiResponse, [
  "user.name.first",
  "user.address.city"
]);

console.log(picked);

/*{
  user: {
    name: { first: "Alicia" },
    address: { city: "Austin" }
  }
}*/


```



------

## DTO Mapper Utility

This small helper lets you **convert any API response into a clean DTO** using a simple mapping definition.  

You define *what you want*, and the function picks the correct values—even from deep paths—and can transform them if needed.

---

### Why Use This?

APIs often return messy or deeply nested data.  
This utility helps you:

- Pick only the fields you need  
- Map deep paths easily (`user.name`, `meta.created`, etc.)  
- Apply optional transformations  
- Build consistent DTOs across your app  
- Keep code clean and reusable  

---

###  How the Mapper Works
The mapper can contain:
1. **String** → treated as a deep path  
2. **Object with `path` + `transform`**  
3. **Nested mapper objects** (for structured DTOs)

---

### Input Data
```js
const response = {
  user: { name: "ayaan", age: null, skills: ["java", "node", "html"] },
  meta: { created: "12/12/2012" }
};

const mapper = { name: "user.name", 
               age: { path: "user.age", transform: (v) => v || 10 }, 
               userSkill: { 
			             path:  "user.skills",
                         transform: (v) => v.map((item) => item.toUpperCase())
                         },
               meta: { 
                  metaCreated: { 
                            path: "meta.created", 
                            transform: (v) => new Date(v).toISOString()
                           } 
                       } 
                   };


dto(response, mapper);
/*
{
  name: "ayaan",
  age: 10, 
  userSkill: [ 'JAVA', 'NODE', 'HTML' ],
  meta: {
    metaCreated: "2012-12-12T00:00:00.000Z"
  }
}
*/
```

Happy Coding:

Instagram : [@_codewithayaan](https://www.instagram.com/_codewithayaan/)
Youtube:  [@_codewithayaan](https://www.youtube.com/@_codewithayaan)
