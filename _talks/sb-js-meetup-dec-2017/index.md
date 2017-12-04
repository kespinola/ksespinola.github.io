---
layout: post
title: SB JS Meetup
date: 12-12-07
---

Setting up continuous deployment of a react relay application to amazon using circleci.

[Follow along repo](https://github.com/ksespinola/relay-aws-starter)

## 1. Install packages and listing scripts

```json
{
  "name": "relay-aws-starter",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "antd": "^2.12.3",
    "farce": "^0.2.1",
    "flexboxgrid": "^6.3.1",
    "found": "^0.3.2",
    "found-relay": "next",
    "graphql": "^0.10.5",
    "moment": "^2.19.1",
    "postcss-flexbugs-fixes": "3.0.0",
    "postcss-loader": "2.0.6",
    "promise": "7.1.1",
    "react": "16",
    "react-dev-utils": "^3.0.2",
    "react-dom": "16",
    "react-error-overlay": "^1.0.9",
    "react-flexbox-grid": "^2.0.0",
    "react-relay": "^1.1.0",
    "relay-local-schema": "^0.6.2",
    "relay-runtime": "^1.1.0"
  },
  "devDependencies": {
    "babel-plugin-relay": "^1.1.0",
    "babel-plugin-transform-flow-strip-types": "^6.22.0",
    "babel-preset-react-app": "^3.0.1",
    "concurrently": "^3.5.0",
    "eslint": "^4.8.0",
    "eslint-config-prettier": "^2.6.0",
    "flow-bin": "^0.59.0",
    "flow-webpack-plugin": "^1.2.0",
    "husky": "^0.14.3",
    "lint-staged": "^4.2.3",
    "neutrino": "^7.3.0",
    "neutrino-middleware-env": "^6.1.4",
    "neutrino-preset-jest": "^6.1.4",
    "neutrino-preset-react": "^7.3.0",
    "prettier": "1.7.3",
    "relay-compiler": "^1.1.0"
  },
  "scripts": {
    "start": "GRAPHQL_URL=$GRAPHQL_URL neutrino start",
    "build": "neutrino build",
    "test": "neutrino test",
    "relay": "relay-compiler --src ./src --schema ./schema.graphql",
    "dev": "concurrently 'yarn relay:watch' 'yarn start'",
    "relay:watch": "npm run relay -- --watch",
    "schema": "node schema.js",
    "precommit": "lint-staged",
    "format": "prettier --write"
  },
  "lint-staged": {
    "*.{js,json,css}": ["npm run format", "git add"]
  },
}
```

## 2. Setting up relay

Responsibilities:

- Build static json query objects used by the relay-runtime to make requests. See git ignored output in `__generated__` folders.

- Validate queries written against the projects `schema.graphql`

### Generating `schema.graphql`

Here is how to generate a `schema.grapqhl` file from any valid graphql endpoint using curl and a script.

_Many of the backend graphql libraries will have a command for generating this file._

#### a. curl for `schema.json`

```bash
curl -X POST \
"https://graphql.myshopify.com/api/graphql" \
-H "Content-Type: application/graphql" \
-d '
query IntrospectionQuery {
   __schema {
     queryType { name }
     mutationType { name }
     subscriptionType { name }
     types {
       ...FullType
     }
     directives {
       name
       description
       locations
       args {
         ...InputValue
       }
     }
   }
 }

 fragment FullType on __Type {
   kind
   name
   description
   fields(includeDeprecated: true) {
     name
     description
     args {
       ...InputValue
     }
     type {
       ...TypeRef
     }
     isDeprecated
     deprecationReason
   }
   inputFields {
     ...InputValue
   }
   interfaces {
     ...TypeRef
   }
   enumValues(includeDeprecated: true) {
     name
     description
     isDeprecated
     deprecationReason
   }
   possibleTypes {
     ...TypeRef
   }
 }

 fragment InputValue on __InputValue {
   name
   description
   type { ...TypeRef }
   defaultValue
 }

 fragment TypeRef on __Type {
   kind
   name
   ofType {
     kind
     name
     ofType {
       kind
       name
       ofType {
         kind
         name
         ofType {
           kind
           name
           ofType {
             kind
             name
             ofType {
               kind
               name
               ofType {
                 kind
                 name
               }
             }
           }
         }
       }
     }
   }
 }' -o schema.json
```
#### b. Run schema script

1. Read in `schema.json`
2. Translate to graphql using `graphql` package
3. Write to `schema.graphql`

```bash
$ npm run schema
```

```javascript
0 const fs = require("fs");
1 const path = require("path");
2 const { buildClientSchema } = require("graphql/utilities/buildClientSchema");
3 const { printSchema } = require("graphql/utilities/schemaPrinter");
4
5 const schemaPath = path.join(process.cwd(), "schema.json");
6
7 fs.readFile(schemaPath, { encoding: "utf-8" }, function(err, file) {
8   const { data } = JSON.parse(file);
9
10   const schema = buildClientSchema(data);
11
12   fs.writeFile("schema.graphql", printSchema(schema), function(err) {
13     if (err) {
14       return console.log(err);
15     }
16
17     console.log("The file was saved!");
18   });
19 });
```

## 3. Project structure

```bash
.
├── .circleci
│   └── config.yml
├── .env
├── .env.sample
├── .eslintrc.json
├── .flowconfig
├── .gitignore
├── .neutrinorc.js
├── README.md
├── package-lock.json
├── package.json
├── public
│   ├── favicon.ico
│   ├── index.html
│   └── manifest.json
├── schema.graphql
├── schema.js
├── schema.json
└── src
    ├── __generated__
    │   ├── routes_App_Query.graphql.js
    │   ├── routes_Catalog_Query.graphql.js
    │   ├── routes_Home_Query.graphql.js
    │   └── routes_ProductShow_Query.graphql.js
    ├── __tests__
    ├── components
    │   └── Product
    │       ├── Product.js
    │       ├── __generated__
    │       │   ├── ProductMutation.graphql.js
    │       │   └── Product_product.graphql.js
    │       ├── index.js
    │       └── product.css
    ├── index.js
    ├── main.js
    ├── routes.js
    ├── screens
    │   ├── App
    │   │   ├── App.js
    │   │   ├── __generated__
    │   │   │   └── App_shop.graphql.js
    │   │   ├── app.css
    │   │   └── index.js
    │   ├── Catalog
    │   │   ├── Catalog.js
    │   │   ├── __generated__
    │   │   │   └── Catalog_shop.graphql.js
    │   │   └── index.js
    │   ├── Collection
    │   │   ├── Collection.js
    │   │   ├── __generated__
    │   │   │   └── Collection_collection.graphql.js
    │   │   └── index.js
    │   ├── Home
    │   │   ├── Home.js
    │   │   ├── __generated__
    │   │   │   └── Home_shop.graphql.js
    │   │   └── index.js
    │   └── ProductShow
    │       ├── ProductShow.js
    │       ├── __generated__
    │       │   └── ProductShow_node.graphql.js
    │       └── index.js
    └── styles
        ├── app.css
        ├── font-awesome.min.css
        └── fonts
            ├── FontAwesome.otf
            ├── fontawesome-webfont.eot
            ├── fontawesome-webfont.svg
            ├── fontawesome-webfont.ttf
            ├── fontawesome-webfont.woff
            └── fontawesome-webfont.woff2
```

## 4. Configure continuous deployment tooling

### CircleCI

#### a. configuration file

```yaml
# .circleci/config.yml

version: 2
jobs:
  build:
    working_directory: ~/app
    docker:
      - image: circleci/node
    steps:
      - checkout
      - restore_cache:
          key: npm-deps-{ checksum "package-lock.json" }
      - run: npm i
      - save_cache:
          key: npm-deps-{ checksum "package-lock.json" }
          paths:
            - node_modules
      - run: npm run relay
      - run: PUBLIC_PATH=$PUBLIC_PATH npm run build
      - persist_to_workspace:
          root: ~/app
          paths:
            - build

  deploy:
    working_directory: ~/app
    docker:
      - image: circleci/python
    steps:
      - attach_workspace:
          at: ~/app
      - run: sudo pip install awscli
      - run: aws s3 sync build s3://$CIRCLE_BRANCH-app --delete --region us-west-2
      - run: aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths /index.html

workflows:
  version: 2

  bd:
    jobs:
      - build
      - deploy:
          requires:
            - build
          filters:
            branches:
              only:
                - master
```

#### b. setup projects

![Circle env]({{ "/assets/images/circleci-project-setup-screenshot.png" | absolute_url }})

#### c. add environment variables

![Circle env]({{ "/assets/images/circleci-env-screenshot.png" | absolute_url }})

[Circleci](https://circleci.com/docs/2.0/env-vars/)

[Travis](https://docs.travis-ci.com/user/environment-variables/)

### AWS

#### a.IAM user

![]({{ "/assets/images/iam-setup-1.png" | absolute_url }})

![]({{ "/assets/images/iam-setup-2.png" | absolute_url }})

![]({{ "/assets/images/iam-setup-3.png" | absolute_url }})

#### b. S3 bucket

![]({{ "/assets/images/aws-s3-bucket-setup.png" | absolute_url }})

#### b. Cloudfront distribution

![]({{ "/assets/images/aws-cloudfront-info.png" | absolute_url }})
