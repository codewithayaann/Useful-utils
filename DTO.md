
## DTO(Data Transfer Object)

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

/***
dto: Maps a data object to a DTO using a mapping definition *
@param {Object} data - The API response or source object *
@param {Object} mapper - Mapping definition * @returns {Object} - Transformed DTO */

function dto(data, mapper) {
  const result = {};
  for (const key in mapper) {
    const rule = mapper[key]; // Case 1: String mapping (deep path)
    if (typeof rule === "string") {
      result[key] = get(data, rule);
    }
   // Case 2: Object with path and optional transform
    else if (rule.path) {
      const rawValue = get(data, rule.path);
      result[key] = rule.transform ? rule.transform(rawValue, data) : rawValue;
    }
    // Case 3: Nested object mapping
    else if (typeof rule === "object") {
      result[key] = dto(data, rule);
    }
  }
  return result;
}


const response = {
  user: { name: "ayaan", age: null, skills: ["java", "node", "html"] },
  meta: { created: "12/12/2012" }
};

const mapper = {
  name: "user.name",
  age: {
    path: "user.age",
    transform: (v) => v || 10
  },

  userSkill: {
    path: "user.skills",
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

