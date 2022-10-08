# Microservice Workshop Day 2, Exercises

## Exercise 0: Tools of the Trade
Install mist-cloud tools: https://mist-cloud.notion.site/Installing-mist-cli-Novice-c2c5662922094f57af8a9eac5204f919

## Exercise 1: A Simple Service
1. Make a product service
    ```json
    // Default/service1/mist.json
    {
      "hooks": {
        "product/view-product": "product-service",
      }
    }
    ```

    ```ts
    // Default/service1/app.ts
    import {
      mistService,
      Envelope,
      postToRapid,
    } from "@mist-cloud-eu/mist-tools-ts";
    import { PRODUCTS } from "./PRODUCTS";

    mistService({
      "product-service": (envelope: Envelope<{ productId: string, userId: string }>) => {
        let product = PRODUCTS.find((x) => x.id === envelope.payload.productId);
        postToRapid("reply", product);
      },
    });
    ```

1. Register view-product event

    ```json
    // events/mist.json
    {
      "view-product": { "waitFor": 5000, "replyCount": 1 }
    }
    ```

3. In one git-bash start the simulator with:
  
    ```sh
    mist run
    ```

4. In another git-bash run this code, which will send a test-request every 30 seconds:

    ```sh
    while true; do curl --silent -X POST \
        -H "Content-Type: application/json" \
        -d '{ "productId": "1", "userId": "0" }' \
        http://localhost:3000/rapid/view-product; echo ""; sleep 30; done
    ```

## Exercise 2: Adding More Services
1. Add a user-service which posts the users preferred location back to the rapid
    <details>
      <summary>Spoiler</summary>

    ```ts
    "user-service": (
      envelope: Envelope<{ productId: string; userId: string }>
    ) => {
      let user = USERS.find((x) => x.id === envelope.payload.userId);
      postToRapid("find-stock", {
        locationId: user?.preferredLocation,
        productId: envelope.payload.productId,
      });
    },
    ```
    </details>
2. Add a stock-service which runs on the event from the user-service and replies with the stock
    <details>
      <summary>Spoiler</summary>

    ```ts
    "stock-service": (
      envelope: Envelope<{ locationId: string; productId: string }>
    ) => {
      let stock = STOCK.find(
        (x) =>
          x.location === envelope.payload.locationId &&
          x.product === envelope.payload.productId
      );
      postToRapid("reply", {
        stock: stock?.stock,
      });
    },
    ```
    </details>
3. Add hooks for the two new services
    <details>
      <summary>Spoiler</summary>

    ```json
    {
      "hooks": {
        "product/view-product": "product-service",
        "user/view-product": "user-service",
        "stock/find-stock": "stock-service"
      }
    }
    ```
    </details>
4. Why did our output not change?
5. Update the event registration
    <details>
      <summary>Spoiler</summary>

    ```json
    {
      "view-product": { "waitFor": 5000, "replyCount": 2 }
    }
    ```
    </details>

## Exercise 3: Welcome to Gitlab
1. Create a gitlab account
2. Create a blank project in gitlab (_without_ README)
3. Click clone and copy the https url
4. In the service run the commands
    ```sh
    > git init
    > git remote add origin [clone url from step 3]
    > git add .
    > git commit -m "Initial commit"
    > git push origin HEAD
    ```
5. Close vscode
5. In a new location (like your desktop) run the command:
    ```
    git clone [clone url from step 3]
    ```
6. Open vscode from this new location

# Exercise 4: Continuous Integration
1. Run the command
    ```shell
    npm i --save-dev typescript
    ```
2. Add a compile script to package.json
    <details>
      <summary>Spoiler</summary>

    ```json
    // package.json
    "scripts": {
      "start": "node app.js",
      "compile": "tsc",
      "test": "echo \"Error: no test specified\" && exit 1"
    },
    ```
    </details>
3. Add a `.gitlab-ci.yml` file
    ```
    image: node:latest

    👷‍♂️:
      stage: build
      script:
        - npm ci
        - npm run compile
    ```
4. Commit and push your changes
5. Did your pipeline succeed?
6. Make it fail
7. Fix it again

## Exercise 5: Safer Integration
1. Extract the product service to a method
    <details>
      <summary>Spoiler</summary>

    ```ts
    // app.ts
    function productService() {
      return (envelope: Envelope<{ productId: string }>) => {
        let product = PRODUCTS.find((x) => x.id === envelope.payload.productId);
        postToRapid("reply", product);
      };
    }

    mistService({
      "product-service": productService(),
      ...
    ```
    </details>
2. Encapsulate PRODUCTS in an airlock
    <details>
      <summary>Spoiler</summary>

    ```ts
    // PRODUCTS.ts
    const PRODUCTS = [
      { id: "0", name: "Coffee", price: 5 },
      { id: "1", name: "Monster", price: 20 },
      { id: "2", name: "Mokai", price: 20 },
    ];

    export class ProductsAirlock {
      fetch(id: string) {
        return PRODUCTS.find((x) => x.id === id);
      }
    }
    ```

    ```ts
    // app.ts
    function productService(products: ProductsAirlock) {
      return (envelope: Envelope<{ productId: string }>) => {
        let product = products.fetch(envelope.payload.productId);
        postToRapid("reply", product);
      };
    }

    mistService({
      "product-service": productService(new ProductsAirlock()),
      ...
    ```
    </details>
3. Extract interface from implementation
    <details>
      <summary>Spoiler</summary>

    ```ts
    // PRODUCTS.ts
    export interface ProductsAirlock {
      fetch(id: string): {id: string, name: string, price: number} | undefined;
    }
    export class RealProductsAirlock implements ProductsAirlock {
      fetch(id: string) {
        return PRODUCTS.find((x) => x.id === id);
      }
    }
    ```
    </details>
4. Install `jest`
    ```sh
    > npm i --save-dev jest
    > npm i --save-dev @types/jest
    ```
5. Make an integration test
    ```ts
    // app.spec.ts
    jest.mock("@mist-cloud-eu/mist-tools-ts");
    const postToRapidMock = postToRapid as jest.MockedFunction<typeof postToRapid>;

    test("Integration test #slow", () => {
      postToRapidMock.mockReset();
      productService(new RealProductsAirlock())({
        messageId: "",
        traceId: "",
        payload: { productId: "0" },
      });
      expect(postToRapid).toHaveBeenCalled();
      expect(postToRapid).toHaveBeenCalledWith("reply", {
        id: "0",
        name: "Coffee",
        price: 5,
      });
    });
    ```
6. Add a stub
    <details>
      <summary>Spoiler</summary>

    ```ts
    // app.spec.ts
    import { ProductsAirlock } from "./PRODUCTS";

    class ProductsAirlockDummy implements ProductsAirlock {
      fetch(id: string): ReturnType<ProductsAirlock["fetch"]> {
        throw "Should not be called";
      }
    }
    class ProductsAirlockStub extends ProductsAirlockDummy {
      fetch(id: string) {
        return { id: "id", name: "name", price: Number.NaN };
      }
    }
    ```
    </details>
7. Make a unit test
    <details>
      <summary>Spoiler</summary>

    ```ts
    // app.spec.ts
    test("Unit test #fast", () => {
      postToRapidMock.mockReset();
      productService(new ProductsAirlockStub())({
        messageId: "",
        traceId: "",
        payload: { productId: "0" },
      });
      expect(postToRapid).toHaveBeenCalled();
      expect(postToRapid).toHaveBeenCalledWith("reply", {
        id: "id",
        name: "name",
        price: Number.NaN,
      });
    });
    ```
    </details>
8. Add fast tests to pipeline
    ```json
    // package.json
    "scripts": {
      "start": "node app.js",
      "compile": "tsc",
      "test": "jest \".*\\.js\""
    },
    ```

    ```yml
    # .gitlab-ci.yml
    image: node:latest

    👷‍♂️:
      stage: build
      script:
        - npm ci
        - npm run compile

    👩‍🔬:
      stage: test
      script:
        - npm run test -- -t=#fast
    ```

## Exercise 6: Toggling Features
1. Make a new ProductsAirlock class that just returns a constant
1. Make a new `FeatureToggles`-class
    <details>
      <summary>Spoiler</summary>

    ```ts
    // FeatureToggles.ts
    export class FeatureToggles {
      static TicketName1() {
        return false;
      }
    }
    ```
    </details>
3. Change the product service so it either uses the new or the old ProductAirlock based on the feature flag
4. Commit and push your changes
5. Did the pipeline pass?

## Exercise 7: Continuous Delivery
8. Add slow tests to pipeline
    <details>
      <summary>Spoiler</summary>
    ```yml
    # .gitlab-ci.yml
    image: node:latest

    👷‍♂️:
      stage: build
      script:
        - npm ci
        - npm run compile

    👩‍🔬:
      stage: test
      script:
        - npm run test -- -t=#fast
    👩‍⚖️:
      stage: staging
      script:
        - npm run test -- -t=#slow
    ```
    </details>

## Exercise 8: Continuous Deployment






