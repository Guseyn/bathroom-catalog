# bathrom-catalog

## 1. Rules Engine

### JSON Schemas

1. Collections

```json
{
  "$schema": "https://json-schema.org",
  "title": "Collection",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "example": "Коллекция Loft"
    }
  },
  "required": ["id", "name"]
}
```

2. Category

```json
{
  "$schema": "https://json-schema.org",
  "title": "Category",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "example": "смеситель для раковины, душевая стойка, полотенцесушитель, раковина и т.д"
    },
    "collectionId": {
      "type": "string",
      "format": "uuid"
    }
  },
  "required": ["id", "name", "collectionId"]
}
```

3. Product

```json
{
  "$schema": "https://json-schema.org",
  "title": "Product",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "sku": {
      "type": "string",
      "example": "3197305"
    },
    "name": {
      "type": "string",
      "example": "смеситель для раковины, душевая стойка, полотенцесушитель, раковина и т.д"
    },
    // Just naive representation for the base cost (without any touches), in real app it can be more flexible and robust
    "baseCost": {
      "type": "string",
      "example": "$45"
    },
    "collectionId": {
      "type": "string",
      "format": "uuid"
    },
    "categoryId": {
      "type": "string",
      "format": "uuid"
    },
    "availableTouchIds": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "uuid"
      }
    }
  },
  "required": ["id", "name", "collectionId", "categoryId", "availableTouchIds"]
}
```

4. Touches

```json
{
  "$schema": "https://json-schema.org",
  "title": "Touch",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "example": "хром, матовый чёрный, золото и др."
    }
  },
  "required": ["id", "name"]
}
```

5. Rules 

```json
{
  "$schema": "https://json-schema.org",
  "title": "Rule",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "example": "Смеситель серии A совместим только с раковинами серии B и C"
    },
    "selectedProductId": {
      "type": "string",
      "format": "uuid"
    },
    "compatibleOnlyProductIds": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "uuid"
      }
    },
    "prohibitedProductIds": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "uuid"
      }
    },
    "listOfProductCombinations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "additionalSelectedProductIds": {
            "type": "array",
            "items": {
              "type": "string",
              "format": "uuid"
            }
          },
          "requiredProductIds": {
            "type": "array",
            "items": {
              "type": "string",
              "format": "uuid"
            }
          }
        },
        "required": ["additionalSelectedProductIds", "requiredProductIds"]
      }
    }
  },
  "required": ["id", "name", "selectedProductId", "compatibleOnlyProductIds", "prohibitedProductIds", "listOfProductCombinations"]
}

```

### Schema relationships (entity diagram)

erDiagram
    Collection ||--o{ Category : contains
    Category ||--o{ Product : contains
    Product }o--o{ Touch : "availableTouchIds"
    Rule ||--|| Product : "selectedProductId"
    Rule }o--o{ Product : "compatibleOnlyProductIds"
    Rule }o--o{ Product : "prohibitedProductIds"
    Rule ||--o{ ProductCombination : "listOfProductCombinations"
    ProductCombination }o--o{ Product : "additionalSelectedProductIds"
    ProductCombination }o--o{ Product : "requiredProductIds"

    Collection {
        uuid id
        string name
    }
    Category {
        uuid id
        string name
        uuid collectionId
    }
    Product {
        uuid id
        string sku
        string name
        string baseCost
        uuid collectionId
        uuid categoryId
        uuid[] availableTouchIds
    }
    Touch {
        uuid id
        string name
    }
    Rule {
        uuid id
        string name
        uuid selectedProductId
        uuid[] compatibleOnlyProductIds
        uuid[] prohibitedProductIds
    }
    ProductCombination {
        uuid[] additionalSelectedProductIds
        uuid[] requiredProductIds
    }

### SimpleRule structure (what each field does)

flowchart LR
    subgraph Rule["SimpleRule"]
        SP[selectedProductId<br/>anchor product]
        CO[compatibleOnlyProductIds<br/>whitelist for extras]
        PR[prohibitedProductIds<br/>blacklist]
        LC[listOfProductCombinations]
    end

    subgraph Combo["Each combination"]
        AS[additionalSelectedProductIds<br/>when user also picks...]
        RQ[requiredProductIds<br/>...must also buy]
    end

    LC --> Combo
    SP --> Engine((Engine))
    CO --> Engine
    PR --> Engine
    LC --> Engine


### Different Types of Rules

1. You select a product by using `selectedProductId`, and you can set the only compatible products for the selected product. It means that you cannot select additional products that is not in `compatibleOnlyProductIds`. Other properties like `prohibitedProductIds`, `listOfProductCombinations` can be empty.
1. There only limited number of touches that we store in a product with `selectedProductId`and products in `compatibleOnlyProductIds` if those are specified. It means that each product in `selectedProductId` and `compatibleOnlyProductIds` can support only those touches that we specify in property `availableTouchIds` for those products.
1. Select product `selectedProductId` has prohibited products, e.i. `prohibitedProductIds`. `compatibleOnlyProductIds` and `listOfProductCombinations` can be empty.
1. Last type of rule is when a user selects additional products aka  `listOfProductCombinations.additionalSelectedProductIds`, and for compination of products `selectedProductId` and `additionalSelectedProductIds` we require to buy products: `requiredProductIds`. In simple case, `additionalSelectedProductIds` is just one element and `requiredProductIds` is one element as it's mentioned in the task. But in real life, it can be more complex, so let's design our system for combination with any number of products.

All these rules must be consistent, meaning `prohibitedProductIds` cannot contains products in `compatibleOnlyProductIds` and vice versa. Also products in `listOfProductCombinations` don't contradict logically products in `prohibitedProductIds` and `compatibleOnlyProductIds`. Otherwise, our Engine throws admin error and requires us to fix the error.

### How Engine Works on Checking compatiblity of selected products and touches

**INPUT**
1. User selects a product with 'input.selectedProductId'.
1. User **may or may not** selects additional products as array of `input.additionalSelectedProductIds`.
1. User selects a touch with 'input.selectedTouchId'.

**ENGINE PROCESS**
1. Engine filters all the rules by `input.selectedProductId` and `input.additionalSelectedProductIds`.
1. Engine validates all those rules individually and all together for like in Conflicts Resolution Section.

For each rule:
1. Enigne gets `availableTouchIds` from product with `input.selectedProductId`.
1. Engine checks all products from `input.additionalSelectedProductIds`, and gets all their `availableTouchIds`.
1. Engine checks all `compatibleOnlyProductIds`. And if there are some in `input.additionalSelectedProductIds` that don't match `compatibleOnlyProductIds`, we inform a user about incompitability or we refuse the order.
1. Engine checks `prohibitedProductIds`. If there any products are matching `input.selectedProductId` and any of `input.additionalSelectedProductIds` we refuse the order.
1. Engine checks `listOfProductCombinations` (if a user selects additional products) and then it checks whether a user required to buy products (`requiredProductIds`). If everything is okay we get their `availableTouchIds`, otherwise we require to add those things to the order.
1. Engine checks if all products in combination support 'input.selectedTouchId'. If yes, the touch is compatible with user order, otherwise no.
1. Also engine can calculate total cost of selected product with touches combined. It's a separate process, and it's out of scope of this task.

### How to Transfer Client Rule to our Schema for Rules.

1. For each input product, we should choose the only compatible products. It does not really matter from which collections those products are from. For example, product **A** from collection **1** potentillay can be only compatible from product **B** in collection **2**. It's just for the flexibility at zero cost.
2. Each product must support certain touches and have a base cost.
3. Each product can have prohibited products, then we paste them in `prohibitedProductIds`.
4. And finally each product if order with certain products must require additional products aka `listOfProductCombinations.requiredProductIds`

flowchart TD
    Start([Client business rule]) --> Q1{Which product<br/>is the anchor?}

    Q1 --> SP[Set selectedProductId]

    SP --> Q2{Which products can<br/>be added with it?}
    Q2 -->|Only specific ones| CO[Fill compatibleOnlyProductIds]
    Q2 -->|Any / not specified| COE[Leave compatibleOnlyProductIds empty]

    SP --> Q3{Any products<br/>must NOT be used?}
    Q3 -->|Yes| PR[Fill prohibitedProductIds]
    Q3 -->|No| PRE[Leave prohibitedProductIds empty]

    SP --> Q4{If user picks X,<br/>must they also buy Y?}
    Q4 -->|Yes| LC[Add to listOfProductCombinations]
    Q4 -->|No| LCE[Leave listOfProductCombinations empty]

    LC --> LC1[additionalSelectedProductIds = X]
    LC --> LC2[requiredProductIds = Y]

    CO --> Q5{Which finishes<br/>does each product support?}
    COE --> Q5
    PR --> Q5
    PRE --> Q5
    LC1 --> Q5
    LC2 --> Q5
    LCE --> Q5

    Q5 --> AT[Set Product.availableTouchIds<br/>on anchor + related products]
    AT --> Q6{Price?}
    Q6 --> BC[Set Product.baseCost]

    BC --> Admin[Run admin conflict checks]
    Admin -->|Pass| Done([Rule ready in catalog])
    Admin -->|Fail| Fix[Fix contradictions]
    Fix --> Admin

### Conflicts Resolution

#### Empty lists mean
1. If `compatibleOnlyProductIds` is empty, the user can pick any additional product (unless it is prohibited).
1. If `prohibitedProductIds` is empty, nothing is blocked by this rule.
1. If `listOfProductCombinations` is empty, this rule does not require any extra products.

#### Individual Rule Check (for admins)
1. A product cannot be in both `compatibleOnlyProductIds` and `prohibitedProductIds`.
1. `selectedProductId` cannot be in `prohibitedProductIds`.
1. `selectedProductId` should not be in `compatibleOnlyProductIds`.
1. Products in `additionalSelectedProductIds` cannot be in `prohibitedProductIds`.
1. Products in `requiredProductIds` cannot be in `prohibitedProductIds`.
1. If `compatibleOnlyProductIds` is not empty, every `additionalSelectedProductIds` entry must be inside `compatibleOnlyProductIds`.
1. The same `additionalSelectedProductIds` set cannot appear twice with different `requiredProductIds`.
1. All product IDs in the rule must exist in the product catalog.

If any check fails, the engine throws an admin error and the rule must be fixed before use.

#### Two or more rules with the same `selectedProductId` (for admin)
1. A product allowed in one rule’s `compatibleOnlyProductIds` cannot be in another rule’s `prohibitedProductIds` for the same `selectedProductId`.
2. If two rules both have `compatibleOnlyProductIds`, their lists must share at least one product. If the overlap is empty, no additional product can satisfy both rules.
3. Two rules cannot require different `requiredProductIds` for the same `additionalSelectedProductIds` set.
4. A product required by one rule cannot be prohibited by another rule for the same `selectedProductId`.

If any check fails, the engine throws an admin error and the rules must be fixed before use.












