# Lab 1.2 — Import APIs and Configure Backends

> **Module 1.3 — Backends & API Importing**

---

## 1. Lab overview

In this lab, you will:

- Import a **real HTTP API** into Azure API Management (APIM) using an **OpenAPI document**.
- Configure and reuse **Backend** resources in APIM.
- Use **Named values** for backend configuration (base URLs, secrets).
- Create a simple **Azure Function HTTP API** and expose it through APIM.
- Observe how **frontend operations** in APIM map to **backend routes** and credentials.

You’ll work with two concrete examples:

1. **Swagger Petstore API** (public HTTP API with an official OpenAPI spec).  
2. A simple **Azure Function** HTTP trigger acting as a small REST endpoint.

By the end of this lab, you should be able to:

- Explain the difference between **APIM APIs**, **Backends**, and **Named values**.
- Import APIs from **OpenAPI specs** and **Azure Functions**.
- Configure and reuse backends across APIs and environments (dev vs prod).

---

## 2. Prerequisites

Before you start, ensure you have:

1. **Completed Lab 1.1 or equivalent setup**  
   - You have an APIM instance (Developer tier or similar) that you can use for this lab.
   - You have **Contributor** (or higher) permission on the APIM resource and Function Apps you’ll create/use.

2. **Azure subscription & resource group**
   - An Azure subscription where you can create resources.
   - A resource group (for example, `rg-apim-training`) that contains your APIM instance and into which you can place the function app you’ll create.

3. **Tools**
   - A modern browser (Edge, Chrome, etc.).
   - Optional: `curl`, Postman, or another HTTP client to call APIs from outside the portal.

4. **APIM instance online**
   - From the Azure portal, confirm your APIM instance shows **Status: Online**.

> ✅ **Checkpoint:** You know **which APIM instance** you’ll use and you can open it in the Azure portal.

---

## 3. Task 1 — Quick review of Backends & Named values in your APIM instance

> **Goal:** Make sure you know where **Backends** and **Named values** live in the APIM UI.

1. In the Azure portal, open your **API Management** instance.
2. In the left menu, under the **APIs** section, click **Backends**.
   - Observe any existing backend entries (you may see none; that’s fine).
   - Note: A backend represents a **real service** APIM can call (HTTP, Function App, Logic App, etc.).
3. In the left menu, click **Named values**.
   - Observe any existing named values.
   - Notice which ones are marked as **Secret** vs **Plain**.

> 💡 **Concept reminder:**
>
> - **Backends** encapsulate a **runtime URL** and related credentials/config.  
> - **Named values** are key/value pairs used across APIs and policies (for base URLs, API keys, connection strings, etc.).
> ✅ **Checkpoint:** You can locate and explain **Backends** and **Named values** in your APIM instance.

---

## 4. Task 2 — Import the Swagger Petstore API via OpenAPI

> **Goal:** Import a public HTTP API using its **OpenAPI (Swagger) document** and create a new API in APIM.

You’ll use the official Swagger **Petstore** OpenAPI **3.0** document, which is frequently used in Microsoft APIM and OpenAPI tutorials.

### 4.1 Locate the OpenAPI 3.0 document URL

For reference, the Petstore **OpenAPI 3.0** JSON endpoint is:

```text
https://petstore3.swagger.io/api/v3/openapi.json
```

> ℹ️ This URL hosts an OpenAPI 3.0 document describing the Petstore API (operations like `/pet`, `/store`, `/user`).

### 4.2 Import the Petstore API into APIM

1. In your APIM instance, go to **APIs**.
2. Click **+ Add API**.
3. Select the **OpenAPI** tile.
4. In the **Create from OpenAPI specification** dialog, choose **Full** (not “Basic”).  
5. Enter the following values:

   - **OpenAPI specification**: choose **From URL** (or similar) and enter:  
     `https://petstore3.swagger.io/api/v3/openapi.json`
   - **Display name**: `Swagger Petstore - OpenAPI 3.0`
   - **Name**: `swagger-petstore` (or accept the default).
   - **API URL suffix**: `petstore`  
     - This means your APIM URL will look like:  
       `https://<your-apim-name>.azure-api.net/petstore`
   - **Scheme**: `HTTPS`.
   - **Subscription required**: **Enabled**.

6. Click **Create** and wait for the API to be imported.

### 4.3 Inspect the imported operations

1. After creation, select the new **Swagger Petstore** API.
2. On the **Design** (or **Operations**) tab, review the list of operations:
   - Examples: `GET /pet/{petId}`, `POST /pet`, `GET /store/inventory`, etc.
3. Pick a couple of interesting operations and note:
   - The **frontend path** under APIM (for example, `/petstore/pet/{petId}`).  
   - The **HTTP method** and parameters.

> ✅ **Checkpoint:** You have a **Petstore API** imported into APIM using **OpenAPI 3.0** and can see multiple operations in the **Design** view.
---

## 5. Task 3 — Create a Backend and Named values for Petstore

> **Goal:** Centralize the Petstore base URL in a **Named value** and use a **Backend** resource so the configuration is reusable and environment‑aware.

### 5.1 Create Named values for Petstore base URLs

1. In your APIM instance, go to **Named values**.
2. Click **+ Add** and create the following plain values:

   1. **Dev base URL**
      - **Name**: `petstore-base-url-dev`
      - **Display name**: `PetstoreBaseURLDev`
      - **Type**: `Plain`
      - **Value**: `https://petstore3.swagger.io/api/v3`
      - **Tags** (optional): `petstore`, `dev`
      - Click **Save**.

   2. **Prod base URL** (for stretch/preview)
      - **Name**: `petstore-base-url-prod`
      - **Display name**: `PetstoreBaseURLProd`
      - **Type**: `Plain`
      - **Value**: `https://petstore3.swagger.io/api/v3`  
        > For this lab, dev and prod use the same host. In a real environment, these would point to different hosts or deployments.
      - **Tags** (optional): `petstore`, `prod`
      - Click **Save**.

> 💡 You can now reference these values anywhere in APIM using the `{{name}}` syntax, for example: `{{PetstoreBaseURLDev}}`.

### 5.2 Connect the Petstore API to the Backend (API-level policy)

You’ll now update the API’s inbound policy so all operations use the `petstore-backend-dev` backend.

1. Go to **APIs** and open your **Swagger Petstore** API.
2. Select the **Design** view and make sure the **API** (top-level) is selected (not an individual operation).
3. Click **</> Code editor** in **Inbound processing**.
4. Replace the existing `<policies>` block (or wrap existing content) so it looks like this:

   ```xml
   <policies>
     <inbound>
       <base />
       <!-- Route all operations on this API to the petstore-backend-dev -->
       <set-backend-service base-url="{{PetstoreBaseURLDev}}" />
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

5. Click **Save**.

> 💡 **Concept link:**
>
> - The **API-level policy** applies to **all operations** in the Petstore API.  
> - `set-backend-service` ties the API to your **backend entity**, which in turn uses a **Named value** for the base URL.
> ✅ **Checkpoint:** Any operation on your Petstore API will now route through the `PetstoreBaseURLDev` URL.

---

## 6. Task 4 — Test Petstore operations and observe frontend vs backend mapping

> **Goal:** Use the APIM **Test** tab to see how the APIM frontend path maps to the backend Petstore URL.

### 6.1 Test a read-only Petstore operation

1. In your APIM instance, open the **Swagger Petstore** API.
2. On the **Design** tab, select the `GET /pet/findByStatus` operation (or another GET operation that doesn’t require a body).
3. Click the **Test** tab.
4. Ensure the **Ocp-Apim-Subscription-Key** header is populated with a valid subscription key.
5. Provide any required parameters, for example:
   - For `GET /pet/findByStatus`, set `status` to `available`.
6. Click **Send**.

### 6.2 Inspect the response and trace

1. Verify you receive a **200 OK** response and a JSON body with Petstore data.
2. If available in your portal experience, turn on **Trace** (or **Debug**), then re‑execute the operation.
3. In the trace output:
   - Look for a line showing the **backend URL** that APIM called.
   - Confirm it matches the expected Petstore endpoint (for example, `https://petstore3.swagger.io/api/v3/pet/findByStatus?...`).

> ✅ **Checkpoint:** You can clearly articulate:
>
> - **Frontend URL** (APIM gateway): `https://<apim-name>.azure-api.net/petstore/...`
> - **Backend URL** (real service): `https://petstore3.swagger.io/api/v3/...`

---

## 7. Stretch Task A — Add more Petstore operations pointing to different backend routes

> **Goal:** Add multiple operations in APIM that all use the same backend but different **paths** / **operations**.

Using your **Swagger Petstore** API:

1. In APIM, open the **Petstore** API and go to the **Design** tab.
2. Identify at least **two additional operations** that you find useful, such as:
   - `GET /store/inventory`
   - `GET /pet/{petId}`
   - `POST /store/order`
3. If they weren’t imported originally, add them manually:
   - Click **+ Add operation**.
   - Configure:
     - **Method**: `GET` or `POST`
     - **URL**: e.g., `/store/inventory` or `/store/order`
     - **Description**: brief description of what it does.
4. Because you added the **API-level `set-backend-service` policy**, new operations automatically point to the `petstore-backend-dev` backend.
5. Use the **Test** tab to call each new operation and verify the responses.

> ✅ **Stretch Checkpoint:** Your Petstore API now includes multiple operations using the same backend, each mapping to different backend routes.

---

## 8. Stretch Task B — Simulate dev vs prod backends with Named values

> **Goal:** Use different **Named values** to prepare for multi-environment deployments.

1. Modify the **Named value** for the prod URL - set it to something that is obviously invalid
2. Decide how you want to switch between dev/prod for the Petstore API. Two common patterns:

   **Pattern 1 — Duplicate API with different backend**
   - Clone the Petstore API (e.g., `Swagger Petstore (prod)`).
   - In the cloned API’s **API-level policy**, change:

     ```xml
     <set-backend-service base-url="{{PetstoreBaseURLDev}}" />
     ```

     to:

     ```xml
     <set-backend-service base-url="{{PetstoreBaseURLProd}}" />
     ```

   **Pattern 2 — Versioning or suffix‑based environments**
   - Keep a single logical API but use **API versions** or different **URL suffixes** (`petstore-dev` vs `petstore-prod`), each with its own backend policy.

3. Test the dev and prod variants and confirm they both work (for this lab they point to the same Petstore, but the pattern is what matters).

> ✅ **Stretch Checkpoint:** You can articulate how Named values + Backends enable **environment‑specific configuration** without rewriting each operation.

---

## 11. Reflection questions

Take a few minutes to reflect (or discuss with a partner):

1. How would you explain the relationship between **APIs**, **Backends**, and **Named values** in APIM?
2. What are the pros and cons of importing an API via **OpenAPI** vs importing directly from **Azure Functions**?
3. How do **Named values** help you separate **environment configuration** from **API design**?
4. In a real project, how would you manage secrets (like function keys or API keys) — where do **Key Vault** and **Named values** fit in?
5. How might you extend today’s setup to include additional backend types (App Service, Logic Apps, AKS, etc.) while keeping your configuration maintainable?
