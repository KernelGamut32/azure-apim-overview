# Lab 1.1 — Provision & Explore an Azure API Management (APIM) Instance

> **Module 1.2 — Creating an APIM Instance & Core Concepts**

## 1. Lab overview

In this lab, you will:

- Create (or validate access to) an Azure API Management (APIM) instance.
- Explore the most important blades and objects in the Azure portal.
- Create a simple **Echo / Hello World** API with a single `GET` operation.
- Test your API via the built‑in **Test** tab and observe the role of the **subscription key**.
- (Stretch) Import a small **public HTTP API** (DummyJSON) as a backend and expose it via APIM.

The lab reinforces the core concepts:

- APIM resource, region, and SKU (Developer tier for non‑production labs).
- Frontend (APIM façade) vs backend (real service).
- APIs, operations, products, subscriptions, backends, named values, and developer portal.
- Where policies can be applied (global, API, operation, product).

---

## 2. Prerequisites

Before you start, ensure the following:

1. **Azure subscription**
   - You have access to an Azure subscription where you are allowed to create resources.
   - You have at least **Contributor** permissions on the subscription or on the target resource group.

2. **Resource group**
   - Either you already have a resource group for the course (for example `rg-apim-training`), or you can create one.

3. **APIM instance**
   - Access to an APIM instance for sandbox purposes
   - For non‑production / training scenarios, the **Developer (v2)** or **Developer** tier is typically used because it includes full feature coverage at lower cost (no SLA).

4. **Tools**
   - A modern browser (Edge, Chrome, or similar).
   - Optional: `curl`, Postman, or another HTTP client to call the APIM endpoint from outside the portal.

---

## 3. Task 1 — Locate an appropriate APIM instance

> **Goal:** Ensure you have a working APIM instance to use for the rest of the course.

### 3.1 Locate an existing APIM instance

1. In the Azure portal search bar, type **“API Management services”**.
2. Select your available APIM instance.
3. Pin it to your **Favorites**:
   - In the left navigation, click the **☆** (star) icon to pin API Management services.
   - Optionally, pin the specific APIM instance to your dashboard.

> ✅ **Checkpoint:** You can open the APIM instance, see its **Overview** blade, and confirm it shows a status of **Online**.

---

## 4. Task 2 — Explore core APIM blades and objects

> **Goal:** Get familiar with the APIM portal experience and terminology.

In the left‑hand menu of your APIM instance, explore the following sections.

### 4.1 Overview & Essentials

1. Select **Overview**.
2. Observe:
   - **Resource group**, **Location**, and **Tier**.
   - The **Gateway URL** (this is the base URL clients will call).
   - The **Publisher email** and **Publisher name**.

> 💡 Discussion prompt: How would you align APIM instances with **dev/test/prod** environments?

### 4.2 APIs

1. Select **APIs** from the left menu.
2. Notice any APIs that were auto‑created, such as an **Echo API** (in some quickstarts) or sample APIs.
3. For each listed API, note:
   - The **Display name** and **Name**.
   - The **URL suffix** that is appended to the APIM gateway URL.
   - The **Type** (HTTP, OpenAPI, SOAP, etc.).

You will come back here to create your own APIs in the next task.

### 4.3 Products

1. Select **Products**.
2. Observe any built‑in products, such as:
   - **Starter**
   - **Unlimited**
3. Click on one of the products and explore:
   - **Included APIs**.
   - **Subscriptions** (whether they require approval, and if they have a subscription key).

> 💡 **Key concept:** Products are how you **bundle APIs** and control **subscription & access** for groups of consumers.

### 4.4 Subscriptions

1. Select **Subscriptions** (often under the APIs section).
2. Note:
   - Any **built‑in subscriptions**, such as *Built‑in all-access subscription*.
   - For each subscription, whether it is **enabled** or **suspended**.
3. Click into a subscription to see the **Primary** and **Secondary** keys.
4. Notice the **scope** of the subscription (All APIs, a specific product, or a specific API).

> 🔐 **Important:** APIM uses **subscription keys** to secure APIs. By default, the key is expected either in the `Ocp-Apim-Subscription-Key` HTTP header or as a `subscription-key` query parameter.

### 4.5 Backends

1. Select **Backends**.
2. If none exist, that’s okay—you’ll create one in the stretch part of the lab.
3. Observe that a backend is essentially a **registered external service** (e.g., `https://dummyjson.com`) that an API can reuse across operations.

### 4.6 Named values

1. Select **Named values**.
2. Note any existing values (they may be used for environment‑specific configuration or secrets).
3. Observe:
   - Whether they are marked as **Secret**.
   - How they can be **referenced in policies** using `{{name}}` syntax.

### 4.7 Developer portal

1. In the left menu, look for **Developer portal** (or **Developer portal > Portal overview / Portal settings**, depending on the tier).
2. If it’s not enabled yet:
   - Follow the prompt to **Enable** the developer portal (if supported by your tier).
3. Click **Developer portal** to open it in a new tab.
4. Explore:
   - The **Home** page.
   - The **API listing**.
   - How a developer would see products and APIs they can subscribe to.

> ✅ **Checkpoint:** You can explain what each of these is: **APIs, Products, Subscriptions, Backends, Named values, Developer portal**.

---

## 5. Task 3 — Create a simple Echo / Hello World API

> **Goal:** Create an API surface in APIM and observe frontend vs backend.

In this task, you’ll create a very simple API that returns a fixed response. This reinforces the **frontend façade** concept—APIM can respond without any real backend service.

### 5.1 Create a new HTTP API

1. In your APIM instance, go to **APIs**.
2. Click **+ Add API**.
3. Choose the **HTTP** tile (create an empty HTTP API).
4. In the **Create an HTTP API** wizard, select **Full** if prompted.
5. Fill out the fields:
   - **Display name:** `Hello World API`
   - **Name:** `hello-world` (auto‑filled is fine)
   - **API URL suffix:** `hello`  
     > The full URL will be: `https://<your-apim-name>.azure-api.net/hello`
   - **Scheme:** `HTTPS`
   - **Subscription required:** keep **enabled**.
6. Leave **Web service URL** empty for now (you’re going to mock the response via policy).
7. Click **Create**.

### 5.2 Add a GET operation

1. After the API is created, you’ll land on its **Design** tab.
2. Under **Frontend**, click **+ Add operation**.
3. Configure the operation:
   - **Display name:** `Get Hello`
   - **Name:** `get-hello`
   - **URL:** `GET /` (leave the template as `/` or `/message` if you prefer).
4. Optionally add a short **Description**, like `Returns a Hello World message from APIM`.
5. Click **Save**.

### 5.3 Configure a simple mock response

You’ll use a policy to return a fixed JSON payload from APIM, without calling any backend.

1. With the `Get Hello` operation selected, click the **Code editor** (or **</>** policy editor) for **Inbound processing**.
2. Wrap existing policies in a `<policies>` block if needed, and replace the content with:

   ```xml
   <policies>
     <inbound>
       <base />
       <return-response>
         <set-status code="200" reason="OK" />
         <set-header name="Content-Type" exists-action="override">
           <value>application/json</value>
         </set-header>
         <set-body>@{
           return new JObject(
             new JProperty("message", "Hello from Azure API Management!"),
             new JProperty("timestamp", DateTime.UtcNow.ToString("o")),
             new JProperty("student", context.Request.Headers.GetValueOrDefault("x-student-name", "anonymous"))
           ).ToString();
         }</set-body>
       </return-response>
     </inbound>
     <backend>
       <base />
     </backend>
     <outbound>
       <base />
     </outbound>
     <on-error>
       <base />
     </on-error>
   </policies>
   ```

3. Click **Save**.

> 💡 **Concept link:** This policy is attached at the **operation** level. You could also attach policies at the **API**, **Product**, or **Global** scope.

---

## 6. Task 4 — Test the API from the Azure portal

> **Goal:** Send a test call through APIM and observe the subscription key requirement.

### 6.1 Use the built‑in Test tab

1. With the `Hello World API` selected, open the `Get Hello` operation.
2. Click the **Test** tab at the top.
3. Make sure:
   - The **HTTP method** is `GET`.
   - The **Request URL** shows your APIM gateway plus `/hello` (or your suffix).
4. Under **Headers**, confirm that the `Ocp-Apim-Subscription-Key` header is pre‑populated with a key. This is your **subscription key**.
5. Optionally, add a header:
   - **Name:** `x-student-name`
   - **Value:** your name (e.g., `Alex`).
6. Click **Send**.

### 6.2 Examine the response

1. In the **Response** section, verify that:
   - The **Status code** is `200 OK`.
   - The **Body** contains JSON similar to:

     ```json
     {
       "message": "Hello from Azure API Management!",
       "timestamp": "2025-11-17T18:00:00.0000000Z",
       "student": "Alex"
     }
     ```

2. Observe:
   - The **timestamp** comes from the policy expression.
   - The **student** value reflects your `x-student-name` header (or `anonymous` if omitted).

> ✅ **Checkpoint:** You have successfully created and tested an APIM operation that responds without a real backend service.

---

## 7. Task 5 — Locate and understand subscription keys

> **Goal:** See how APIM secures APIs with subscription keys and how to call your API from outside the portal.

### 7.1 Inspect product and subscription

1. Go to **Products**.
2. Open the product that includes your `Hello World API` (for example, `Unlimited` or `Starter`):
   - If needed, add your API to the product under **APIs**.
3. From the product, navigate to **Subscriptions** (or go back to the **Subscriptions** blade at the root of APIM).
4. Identify a subscription you can use (for training, the built‑in subscription is often fine).
5. Open the subscription and copy the **Primary key**.

### 7.2 Call the API from your browser or curl

Use the **subscription key** in either a header or query string.

#### Option A — Browser (query string)

1. Build a URL in this format (replace values in `<...>`):

   ```text
   https://<your-apim-name>.azure-api.net/hello?subscription-key=<your-subscription-key>
   ```

2. Paste the URL into your browser and press **Enter**.
3. You should see the JSON response directly in the browser.

#### Option B — curl (header)

From a terminal:

```bash
curl -X GET "https://<your-apim-name>.azure-api.net/hello" \
  -H "Ocp-Apim-Subscription-Key: <your-subscription-key>" \
  -H "x-student-name: Alex"
```

You should see the same JSON response, including your `x-student-name` value.

> 💡 **Concept link:** By default, APIM uses the `Ocp-Apim-Subscription-Key` header or a `subscription-key` query parameter to validate access. This is part of APIM’s **data plane** security model.

---

## 8. Stretch Task — Import a public HTTP backend (DummyJSON)

> **Goal:** Import a more interesting **real HTTP backend** and expose it via APIM as a pass‑through API.

For this stretch task, you’ll use [DummyJSON](https://dummyjson.com/) — a free fake REST API with realistic JSON data for testing and prototyping.

### 8.1 Create a backend for DummyJSON

1. In your APIM instance, go to **Backends**.
2. Click **+ Add**.
3. Configure:
   - **Name:** `dummyjson-backend`
   - **URL:** `https://dummyjson.com`
   - Leave other settings at their defaults.
4. Click **Create**.

### 8.2 Create a new API that uses the DummyJSON backend

1. Go to **APIs**.
2. Click **+ Add API**.
3. Choose the **HTTP** tile again.
4. In the **Create an HTTP API** wizard:
   - **Display name:** `DummyJSON Products API`
   - **Name:** `dummyjson-products`
   - **API URL suffix:** `dummy-products`
   - **Web service URL:** `https://dummyjson.com` (same as backend base URL).
   - Ensure **Subscription required** is enabled.
5. Click **Create**.

### 8.3 Add a GET /products operation

1. On the `DummyJSON Products API` **Design** tab, click **+ Add operation**.
2. Configure:
   - **Display name:** `List products`
   - **Name:** `list-products`
   - **URL:** `GET /products`
   - **Description:** `Returns a list of products from DummyJSON`.
3. Click **Save**.

At this point, APIM will **forward** requests to `https://dummyjson.com/products` and relay the response.

### 8.4 Test the DummyJSON API from the portal

1. Select the `List products` operation.
2. Click the **Test** tab.
3. Ensure the **Ocp-Apim-Subscription-Key** header is populated.
4. Click **Send**.
5. Confirm you receive a `200 OK` response with a JSON payload containing **products**.

### 8.5 Call the DummyJSON API from outside the portal

1. Determine the full URL (replace `<your-apim-name>`):

   ```text
   https://<your-apim-name>.azure-api.net/dummy-products/products
   ```

2. Use `curl` to call it, including your subscription key:

   ```bash
   curl -X GET "https://<your-apim-name>.azure-api.net/dummy-products/products" \
     -H "Ocp-Apim-Subscription-Key: <your-subscription-key>"
   ```

3. You should see a JSON response that looks like real product catalog data.

> 💡 **Concept link:** Here, APIM is a **frontend façade** over a **real backend** (DummyJSON). You could now add **policies** for logging, caching, rate limiting, or response shaping without changing the backend.

---

## 9. Reflection questions

Take a few minutes to think about (or discuss):

1. How would you explain the difference between the **APIM frontend** and the **backend service** to a colleague?
2. Where did you see **subscription keys** in action, and how do they relate to **products** and **subscriptions**?
3. How could you extend the DummyJSON API in APIM to:
   - Add **caching**?
   - Add **rate limits** per subscription?
   - Inject additional **response headers** for observability?

You’ll build on these concepts in later modules when you work with **policies, logging, caching, and automation**.
