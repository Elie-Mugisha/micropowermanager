# MicroPowerManager (MPM) - WhatsApp Channel Plugin
## Engineering Task Master Plan & Team Handbook

This document is the definitive, prioritized implementation blueprint for building the **WhatsApp Business Channel Plugin** for MicroPowerManager (MPM). It is structured specifically for engineers with TypeScript/NestJS notion who are building notion in PHP/Laravel and Vue 2.

---

## Architecture Overview & The Core Mental Model

### The Business Reality
MPM is an open-source CRM and energy-billing platform for rural solar mini-grids (primarily across East & West Africa). 
- Customers purchase prepaid electricity credits.
- Today, MPM transmits the generated 20-digit keypad recharge token via cellular SMS.
- **The Challenge:** Telecom carriers in rural regions often silently drop SMS messages while still billing the operator. SMS is also expensive and lacks brand trust.
- **The Solution:** Deliver token receipts and low-balance warnings over WhatsApp Business API (using Twilio / Meta Cloud API) at a fraction of the cost, while maintaining an ironclad **SMS fallback** safety net so no customer is ever left in the dark.

### Multi-Tenancy Architecture (Database-Per-Tenant)
MPM runs as a multi-tenant platform:
1. **Central Database (`micro_power_manager`):** Stores global system data (registered companies, system users, global list of installable plugins).
2. **Tenant Databases (`tenant` / `DemoCompany_1`):** Every solar company has its own isolated database schema containing customers, meters, transactions, and plugin configurations.
3. **The Golden Rule:** Every table created by this plugin must live inside the **tenant** database using tenant migrations. All models must extend `App\Models\Base\BaseModel` (which dynamically binds the tenant connection). Never query tenant tables from the default central connection.

---

## Prioritized Task Backlog

```
Phase 1: Foundation & Schemas
  ├── Task 1: Plugin Scaffolding & System Enum Registration [Priority: P0]
  ├── Task 2: Multi-Tenant Database Schemas & Eloquent Models [Priority: P0]
  └── Task 3: Tenant Configuration API & Validation Layer [Priority: P0]

Phase 2: Core Messaging & Fallback Engine
  ├── Task 4: WhatsApp Gateway Driver (Twilio Client) [Priority: P0]
  ├── Task 5: SMS Gateway Resolver & Fallback Pipeline [Priority: P0]
  └── Task 6: Operator Test Connection Endpoint ("Ping Service") [Priority: P1]

Phase 3: Frontend Administration Console
  ├── Task 7: Vue 2 Settings Dashboard & Localization [Priority: P1]
  └── Task 8: Interactive Test Panel & Delivery Logs Table [Priority: P1]

Phase 4: Reliability & Hackathon Standouts
  ├── Task 9: Inbound Webhook Resolver & Status Tracking [Priority: P1]
  ├── Task 10: Configurable Fallback Policy Engine [Priority: P2]
  └── Task 11: End-to-End Simulation, Bruno Collection & Submission Docs [Priority: P0]
```

---

### Task 1: Plugin Scaffolding & System Enum Registration
**Priority:** P0 (Blocking all work)  
**Assigned Developer:** Developer 1 (Foundation)

#### 1. Plain English Summary
We run MPM’s built-in plugin generator to automatically create the folder structures for both the backend (PHP) and frontend (Vue). Then, we register a unique number (ID) in MPM's central plugin registry so the system recognizes "WhatsApp" as a valid installable feature.

#### 2. User Story
> **As a** platform developer,  
> **I want to** initialize the WhatsApp plugin directory structure and register its core plugin identifier,  
> **So that** both backend services and frontend components have a dedicated, modular home without polluting core MPM code.

#### 3. Acceptance Criteria
* [ ] The directory `src/backend/app/Plugins/Whatsapp/` exists with standard subdirectories (`Console`, `Http`, `Models`, `Providers`, `Services`, `routes`).
* [ ] The directory `src/frontend/src/plugins/whatsapp/` exists.
* [ ] Constant `public const WHATSAPP_MESSAGING = 33;` is registered in `App\Models\MpmPlugin`.
* [ ] Running `php artisan route:list` inside the backend container exits with code 0.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** Like reserving a physical lot in a neighborhood and putting up a sign that says "WhatsApp House will be built here."
* **Technical:** MPM has a modular plugin loader. Each plugin lives in `app/Plugins/{PluginName}` and declares its own `ServiceProvider`, routes, and controllers. The central database table `mpm_plugins` tracks all available plugins by numeric ID.

#### 5. NestJS & TypeScript Mental Model
* Like generating a new sub-module using Nest CLI:
  ```bash
  nest g module plugins/whatsapp
  ```
  And registering the new module in an enum or constant file:
  ```typescript
  export enum PluginId {
    WHATSAPP_MESSAGING = 33,
  }
  ```

#### 6. Files Involved
* **Modify:** [`src/backend/app/Models/MpmPlugin.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Models/MpmPlugin.php) (Add constant).
* **Generated:** `src/backend/app/Plugins/Whatsapp/` (Backend plugin root).
* **Generated:** `src/frontend/src/plugins/whatsapp/` (Frontend plugin root).
* **DO NOT MODIFY:** Any existing plugin folders (e.g. `AfricasTalking`, `ViberMessaging`, `CalinSmartMeter`).

#### 7. Example Workflow
1. Developer runs Artisan CLI generator:
   ```bash
   php artisan micropowermanager:new-plugin Whatsapp
   ```
2. Developer opens `MpmPlugin.php` and adds:
   ```php
   public const WHATSAPP_MESSAGING = 33;
   ```
3. Verification: Verify directory structure via `ls -la src/backend/app/Plugins/Whatsapp`.

---

### Task 2: Multi-Tenant Database Schemas & Eloquent Models
**Priority:** P0 (Prerequisite for Config API & Gateway)  
**Assigned Developer:** Developer 1 (Foundation)

#### 1. Plain English Summary
We create two new database tables inside each solar company's private database:
1. `whatsapp_configs` to store the company’s Twilio API keys, sender phone number, and ON/OFF switch.
2. `whatsapp_message_logs` to store a history of every message we try to send, whether it succeeded on WhatsApp, or fell back to SMS.
Then we create PHP model classes so our code can easily read and write to these tables.

#### 2. User Story
> **As a** solar company administrator,  
> **I want** my WhatsApp credentials and message history stored securely in my own tenant database,  
> **So that** my sensitive API tokens are completely isolated from other companies sharing the platform.

#### 3. Acceptance Criteria
* [ ] Tenant migration `create_whatsapp_configs_table` creates columns: `id`, `account_sid`, `auth_token`, `from_phone_number`, `fallback_strategy`, `is_enabled`, `timestamps`.
* [ ] Tenant migration `create_whatsapp_message_logs_table` creates columns: `id`, `external_message_id`, `recipient_phone`, `message_body`, `channel_used`, `status`, `error_message`, `timestamps`.
* [ ] Model `App\Plugins\Whatsapp\Models\WhatsappConfig` extends `App\Models\Base\BaseModel`.
* [ ] Model `App\Plugins\Whatsapp\Models\WhatsappMessageLog` extends `App\Models\Base\BaseModel`.
* [ ] `php artisan migrate-tenant {company_id}` runs without error and creates both tables in the tenant schema.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** Setting up filing cabinets in each company's private office. Company A cannot open Company B's filing cabinet.
* **Technical:** MPM uses `BaseModel` which explicitly overrides `$this->setConnection('tenant')`. Central migrations live in `database/migrations/`, but tenant migrations **must** live in `database/migrations/tenant/` and be generated with `php artisan make:migration-tenant`.

#### 5. NestJS & TypeScript Mental Model
* Equivalent to creating TypeORM entities with a dynamic multi-tenant connection resolver:
  ```typescript
  @Entity({ name: 'whatsapp_configs' })
  export class WhatsappConfig extends TenantBaseEntity {
    @PrimaryGeneratedColumn() id: number;
    @Column() accountSid: string;
    @Column() authToken: string;
    @Column() fromPhoneNumber: string;
    @Column({ default: 'immediate' }) fallbackStrategy: string;
    @Column({ default: false }) isEnabled: boolean;
  }
  ```

#### 6. Files Involved
* **Create:** `src/backend/database/migrations/tenant/{timestamp}_create_whatsapp_configs_table.php`
* **Create:** `src/backend/database/migrations/tenant/{timestamp}_create_whatsapp_message_logs_table.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Models/WhatsappConfig.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Models/WhatsappMessageLog.php`
* **DO NOT MODIFY:** Existing central migrations in `src/backend/database/migrations/`.

#### 7. Example Workflow
1. Developer generates tenant migrations:
   ```bash
   php artisan make:migration-tenant create_whatsapp_configs_table
   php artisan make:migration-tenant create_whatsapp_message_logs_table
   ```
2. Developer writes table columns in the migration files using Blueprint schema builder.
3. Developer runs migration on Demo tenant:
   ```bash
   php artisan migrate-tenant 1
   ```
4. Verification: Check MySQL schema via `DESCRIBE whatsapp_configs;` inside tenant DB.

---

### Task 3: Tenant Configuration API & Validation Layer
**Priority:** P0 (Prerequisite for Frontend)  
**Assigned Developer:** Developer 1 (Foundation)

#### 1. Plain English Summary
We build two HTTP endpoints:
1. `GET /api/whatsapp/config` so the frontend settings page can display the current saved credentials.
2. `POST /api/whatsapp/config` so when the admin clicks "Save", the credentials are validated (making sure the SID starts with `AC`, fields aren't blank) and securely updated in the database.

#### 2. User Story
> **As a** frontend application,  
> **I want** authenticated REST endpoints to fetch and save WhatsApp credentials,  
> **So that** operators can configure and toggle the channel from their web browser.

#### 3. Acceptance Criteria
* [ ] Route `GET /api/whatsapp/config` returns JSON with existing tenant settings or empty defaults.
* [ ] Route `POST /api/whatsapp/config` validates request parameters using a FormRequest class.
* [ ] `auth_token` is masked or securely handled in responses.
* [ ] Endpoints require valid tenant authentication (JWT middleware).

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** A safe customer service window where the manager presents their ID, asks to see their current settings, or submits an update form. The clerk rejects the form if required fields are missing.
* **Technical:** Laravel `Controller` paired with `FormRequest` (for input validation) and `JsonResource` (for response serialization). Routes defined in `Plugins/Whatsapp/routes/api.php` under the `['middleware' => ['api']]` group.

#### 5. NestJS & TypeScript Mental Model
* Like a NestJS controller using `class-validator` DTOs:
  ```typescript
  export class UpdateWhatsappConfigDto {
    @IsString() @IsNotEmpty() account_sid: string;
    @IsString() @IsNotEmpty() auth_token: string;
    @IsPhoneNumber() from_phone_number: string;
    @IsBoolean() is_enabled: boolean;
  }
  
  @Controller('api/whatsapp/config')
  @UseGuards(JwtAuthGuard)
  export class WhatsappConfigController {
    @Get() getConfig() { ... }
    @Post() updateConfig(@Body() dto: UpdateWhatsappConfigDto) { ... }
  }
  ```

#### 6. Files Involved
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Controllers/WhatsappConfigController.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Requests/WhatsappConfigRequest.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Resources/WhatsappConfigResource.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Services/WhatsappConfigService.php`
* **Modify:** `src/backend/app/Plugins/Whatsapp/routes/api.php`
* **DO NOT MODIFY:** Core auth middleware (`JwtMiddleware`).

#### 7. Example Workflow
1. Client sends `POST /api/whatsapp/config` with payload.
2. `WhatsappConfigRequest` executes `rules()`:
   * If validation fails: returns HTTP 422 with field-specific errors.
3. Controller delegates to `WhatsappConfigService::updateConfig()`.
4. Service updates tenant row in `whatsapp_configs` via Eloquent.
5. Returns HTTP 200 with `WhatsappConfigResource`.

---

### Task 4: WhatsApp Gateway Driver (Twilio API Integration)
**Priority:** P0 (Core engine)  
**Assigned Developer:** Developer 2 (Engine)

#### 1. Plain English Summary
We write the engine class (`WhatsappGateway.php`) that takes a customer's phone number and message text, formats it for WhatsApp, and makes an HTTP POST request to Twilio's WhatsApp API. If Twilio accepts the message, it returns success; if Twilio rejects it, our driver throws an exception.

#### 2. User Story
> **As an** MPM notification system,  
> **I want** a dedicated gateway driver that communicates with Twilio's WhatsApp API,  
> **So that** outbound notification payloads are correctly transmitted to customer WhatsApp accounts.

#### 3. Acceptance Criteria
* [ ] Class `WhatsappGateway` implements standard gateway method:  
  `public function sendSms(string $body, string $receiver, Sms $sms): void`
* [ ] Gateway reads credentials for the active tenant from `WhatsappConfigService`.
* [ ] Payload formatted according to Twilio requirements: `From=whatsapp:{fromNumber}`, `To=whatsapp:{receiver}`, `Body={body}`.
* [ ] Basic Auth header created with `base64(account_sid:auth_token)`.
* [ ] On HTTP 201: Captures Twilio `MessageSid` and returns cleanly.
* [ ] On HTTP 4xx/5xx: Throws custom `WhatsappMessageNotSentException` containing Twilio's error code.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** The automated postman. It receives a letter, puts it into an envelope labeled "WhatsApp", stamps it with the company’s account number and secret password, and delivers it to Twilio's post office.
* **Technical:** Implements MPM's SMS gateway interface pattern (see [`AfricasTalkingGateway.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Plugins/AfricasTalking/AfricasTalkingGateway.php)). Uses Guzzle (`Illuminate\Support\Facades\Http`) to send an asynchronous/synchronous HTTP POST request to `https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages.json`.

#### 5. NestJS & TypeScript Mental Model
* Like an `@Injectable()` service using NestJS `HttpService` / Axios:
  ```typescript
  @Injectable()
  export class WhatsappGateway {
    constructor(private readonly http: HttpService, private readonly config: WhatsappConfigService) {}
    
    async sendSms(body: string, receiver: string, smsId: number): Promise<string> {
      const { accountSid, authToken, fromNumber } = await this.config.get();
      const response = await firstValueFrom(
        this.http.post(`https://api.twilio.com/...`, new URLSearchParams({
          From: `whatsapp:${fromNumber}`,
          To: `whatsapp:${receiver}`,
          Body: body,
        }), {
          auth: { username: accountSid, password: authToken }
        })
      );
      return response.data.sid;
    }
  }
  ```

#### 6. Files Involved
* **Create:** `src/backend/app/Plugins/Whatsapp/WhatsappGateway.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Exceptions/WhatsappMessageNotSentException.php`
* **Modify:** `src/backend/app/Plugins/Whatsapp/Providers/WhatsappServiceProvider.php` (Bind gateway to IoC container).
* **Reference Only:** `src/backend/app/Plugins/AfricasTalking/AfricasTalkingGateway.php` (Inspect as design model).

#### 7. Example Workflow
1. Driver receives: `body: "Your token is 1234-5678"`, `receiver: "+250788123456"`.
2. Driver queries tenant config: verifies `is_enabled == true`.
3. Dispatches HTTP POST to Twilio with basic auth.
4. Twilio replies `{ "sid": "SM999", "status": "queued" }`.
5. Driver completes execution without errors.

---

### Task 5: Gateway Resolver & SMS Fallback Pipeline
**Priority:** P0 (Core differentiator & safety net)  
**Assigned Developer:** Developer 2 (Engine)

#### 1. Plain English Summary
We connect our WhatsApp gateway into MPM's core messaging router (`SmsGatewayResolverService`). Most importantly, we build the **circuit breaker / fallback**: if WhatsApp delivery fails for any reason (bad number, customer not on WhatsApp, API timeout), the system catches the error, writes a log note, and **immediately fires a normal SMS text message** using AfricasTalking or TextBee so the customer always gets their token.

#### 2. User Story
> **As an** off-grid energy customer,  
> **I want** my payment token sent via SMS if WhatsApp fails or my data is off,  
> **So that** I never sit in the dark waiting for electricity.

#### 3. Acceptance Criteria
* [ ] Constant `public const WHATSAPP_GATEWAY = 'WhatsappGateway';` added to `SmsGatewayResolverService`.
* [ ] `SmsGatewayResolverService::getAvailableSmsGateways()` lists WhatsApp when plugin is active.
* [ ] `SmsGatewayResolverService::determineGateway()` picks WhatsApp if configured and enabled in settings.
* [ ] In `WhatsappGateway::sendSms()`, execution is wrapped in a `try/catch (\Throwable $e)`.
* [ ] In `catch`:
  * Record log in `whatsapp_message_logs` with status `fallback_dispatched`.
  * Resolve active SMS gateway (AfricasTalking/TextBee).
  * Dispatch token via fallback SMS gateway.
  * System does not crash or leave the `sms` database row in an unhandled state.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** A flight booking system: Try seat on Airline A (WhatsApp). If the flight is cancelled or full, instantly rebook on Airline B (SMS) without making the traveler do anything.
* **Technical:** Decorator/Fallback pattern inside MPM's notification dispatch pipeline. Intercepts failures at the gateway driver boundary, transitions state in `whatsapp_message_logs`, and delegates execution to the secondary gateway resolver.

#### 5. NestJS & TypeScript Mental Model
* Like an exception filter or fallback interceptor:
  ```typescript
  try {
    await this.whatsappGateway.send(payload);
  } catch (error) {
    this.logger.warn(`WhatsApp delivery failed: ${error.message}. Triggering SMS fallback.`);
    await this.logService.recordFallback(payload, error);
    await this.smsGateway.send(payload); // Fallback execution
  }
  ```

#### 6. Files Involved
* **Modify:** [`src/backend/app/Services/SmsGatewayResolverService.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Services/SmsGatewayResolverService.php)
* **Modify:** `src/backend/app/Plugins/Whatsapp/WhatsappGateway.php`
* **Reference Only:** [`src/backend/app/Jobs/SmsProcessor.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Jobs/SmsProcessor.php)
* **Reference Only:** [`src/backend/app/Services/SmsService.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Services/SmsService.php)

#### 7. Example Workflow
1. Payment completes -> `SendTransactionConfirmationSmsListener` fires.
2. `SmsGatewayResolverService` returns `WhatsappGateway`.
3. `WhatsappGateway` attempts Twilio call.
4. Twilio returns HTTP 400: `21211 (Invalid 'To' Phone Number)`.
5. Gateway catches exception -> writes `whatsapp_message_logs` entry (`status: fallback_dispatched`).
6. Gateway invokes `AfricasTalkingGateway->sendSms(...)`.
7. Customer receives standard cellular SMS.

---

### Task 6: Operator Test Connection Endpoint ("Ping Service")
**Priority:** P1 (Essential for UX & Hackathon Demo)  
**Assigned Developer:** Developer 2 (Engine)

#### 1. Plain English Summary
We build a dedicated backend endpoint (`POST /api/whatsapp/test-connection`) that lets an operator send a test message to their own personal phone number. It tests their saved Twilio credentials live without creating a fake customer or charging a real meter.

#### 2. User Story
> **As a** solar company manager,  
> **I want** to send a test WhatsApp message to my own phone from the settings page,  
> **So that** I have 100% confidence my Twilio account is configured properly before turning the channel ON for real customers.

#### 3. Acceptance Criteria
* [ ] Route `POST /api/whatsapp/test-connection` registered in `routes/api.php`.
* [ ] Accepts `{ "test_phone_number": "+250..." }`.
* [ ] Validates phone number format.
* [ ] Dispatches a sandbox test message: `"Hello from MicroPowerManager! Your WhatsApp Channel integration is working successfully."`
* [ ] Returns JSON `{ "success": true, "message_sid": "SM..." }` on success.
* [ ] Returns JSON `{ "success": false, "error": "Twilio error description" }` with appropriate HTTP status on failure.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** A "Test Sound" button on a speaker before starting the concert.
* **Technical:** An isolated controller action that bypasses the billing pipeline, resolves active tenant credentials, and invokes `WhatsappGateway` with a synthetic payload.

#### 5. NestJS & TypeScript Mental Model
* Equivalent to a health-check action:
  ```typescript
  @Post('test-connection')
  async testConnection(@Body('test_phone_number') phone: string) {
    return await this.whatsappService.sendDiagnosticPing(phone);
  }
  ```

#### 6. Files Involved
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Controllers/WhatsappTestController.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Requests/WhatsappTestRequest.php`
* **Modify:** `src/backend/app/Plugins/Whatsapp/routes/api.php`

#### 7. Example Workflow
1. Operator types `+250788111222` and clicks "Send Test".
2. Controller receives payload, executes Twilio ping via Gateway.
3. Message arrives on operator's phone in 2 seconds.
4. Controller returns `200 OK` with Message SID.

---

### Task 7: Vue 2 Settings Dashboard & Localization
**Priority:** P1 (Frontend Core)  
**Assigned Developer:** Developer 3 (Frontend)

#### 1. Plain English Summary
We build the visual settings page in the MPM admin dashboard. The page gives the manager a clean form to enter their Twilio Account SID, Auth Token, Sender Number, an Enable/Disable switch, and a choice of Fallback Strategy. Every piece of text must come from language translation files (no hardcoded English).

#### 2. User Story
> **As an** operator,  
> **I want** an intuitive settings screen in the MPM navigation sidebar,  
> **So that** I can configure, enable, or disable WhatsApp notifications without writing code.

#### 3. Acceptance Criteria
* [ ] Route `/whatsapp` registered in `src/frontend/src/ExportedRoutes.js` and visible in sidebar.
* [ ] Form fields bound via `v-model`: `account_sid`, `auth_token`, `from_phone_number`, `is_enabled`, `fallback_strategy`.
* [ ] Page fetches current values on `mounted()` via `GET /api/whatsapp/config`.
* [ ] "Save Changes" button submits `POST /api/whatsapp/config` with toast notifications for success/failure.
* [ ] All labels use `$t('words...')` or `$t('phrases...')`.
* [ ] Translations registered in `src/frontend/src/assets/locales/en.json`.
* [ ] `npm run i18n:check-json` passes in frontend container.

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** The dashboard control panel inside the car. The driver toggles switches and adjusts dials with clear labels.
* **Technical:** Vue 2 single-file component (SFC) styled with BootstrapVue components (`<b-form-input>`, `<b-form-checkbox>`, `<b-card>`). Route registered with `meta: { sidebar: { enabled_by_mpm_plugin_id: 33 } }`.

#### 5. NestJS / Frontend Mental Model
* Equivalent to an Angular/React admin settings page connected via Axios:
  ```html
  <template>
    <b-card :title="$t('phrases.whatsappSettings')">
      <b-form-group :label="$t('phrases.accountSid')">
        <b-form-input v-model="form.account_sid" />
      </b-form-group>
      <b-button @click="save" variant="primary">{{ $t('words.save') }}</b-button>
    </b-card>
  </template>
  ```

#### 6. Files Involved
* **Create:** `src/frontend/src/plugins/whatsapp/modules/Overview/Overview.vue`
* **Create:** `src/frontend/src/plugins/whatsapp/services/WhatsappService.js` (Axios API calls)
* **Modify:** [`src/frontend/src/ExportedRoutes.js`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/frontend/src/ExportedRoutes.js)
* **Modify:** `src/frontend/src/assets/locales/en.json` (and other locale files)
* **DO NOT MODIFY:** Core Vue router bootstrap files.

#### 7. Example Workflow
1. Operator navigates to `Settings -> WhatsApp` in MPM sidebar.
2. `Overview.vue` loads credentials from backend.
3. Operator updates keys and flips toggle to `Enabled`.
4. Clicks "Save" -> Toast alert appears: "Settings saved successfully".

---

### Task 8: Interactive Test Panel & Delivery Logs Table
**Priority:** P1 (Frontend Standout Feature)  
**Assigned Developer:** Developer 3 (Frontend)

#### 1. Plain English Summary
On the same settings page, we add two high-value features:
1. A **Test Connection Box**: An input to type a phone number and a `[Send Test Message]` button with a live spinner and success/error alert.
2. A **Message Logs Table**: A table at the bottom showing recent messages, recipient numbers, whether it went through WhatsApp or SMS, and color-coded status badges (Green = Delivered, Blue = Fell back to SMS, Red = Failed).

#### 2. User Story
> **As an** operator,  
> **I want** to see a live log of outgoing WhatsApp messages and their fallback statuses,  
> **So that** when a customer calls support asking about their token, I can instantly see what channel delivered it.

#### 3. Acceptance Criteria
* [ ] Component `TestPanel.vue` provides input for test phone number and triggers `POST /api/whatsapp/test-connection`.
* [ ] Shows green alert with Message SID on success, or red alert with error description on failure.
* [ ] Component `LogsTable.vue` displays paginated logs from `GET /api/whatsapp/logs`.
* [ ] Status rendered with color-coded badges:
  * 🟢 Green: `Delivered (WhatsApp)`
  * 🟡 Yellow: `Sent / In-Flight`
  * 🔵 Blue: `Fallback (SMS)`
  * 🔴 Red: `Failed`

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** An order tracking screen (like tracking a delivery package) right below the settings form.
* **Technical:** A reusable sub-component utilizing `<b-table>` with scoped slots for custom badge rendering and pagination controls.

#### 5. Files Involved
* **Create:** `src/frontend/src/plugins/whatsapp/modules/Overview/Components/TestPanel.vue`
* **Create:** `src/frontend/src/plugins/whatsapp/modules/Overview/Components/LogsTable.vue`
* **Modify:** `src/frontend/src/plugins/whatsapp/modules/Overview/Overview.vue` (Embed components)
* **Modify:** `src/frontend/src/assets/locales/en.json` (Add badge and table translation keys)

---

### Task 9: Inbound Webhook Resolver & Status Tracking
**Priority:** P1 (Delivery Reliability)  
**Assigned Developer:** Developer 4 (Webhooks & QA)

#### 1. Plain English Summary
When WhatsApp delivers a message to a customer's phone, Twilio calls a webhook URL on our server to say: *"Message SM12345 was delivered"*. We build the public endpoint that listens for this call and updates the status in our database from `Sent` to `Delivered`. Because Twilio doesn't have an MPM login account, we use MPM's `ThirdPartyApiResolver` to figure out which company the webhook belongs to.

#### 2. User Story
> **As an** MPM system,  
> **I want** to receive delivery receipts from Twilio's webhook callbacks,  
> **So that** message log statuses accurately reflect whether the customer's handset received the message.

#### 3. Acceptance Criteria
* [ ] Prefix `api/whatsapp/` registered in `ThirdPartyApiResolverService`.
* [ ] Resolver `WhatsappApiResolver` implements `IApiResolver` to resolve `company_id`.
* [ ] Endpoint `POST /api/whatsapp/webhook/{companyId}` accepts Twilio status callbacks.
* [ ] Controller updates `whatsapp_message_logs` record matching `external_message_id = MessageSid`.
* [ ] Status transitions correctly: `queued` -> `sent` -> `delivered` (or `undelivered` / `failed`).

#### 4. Non-Technical vs. Technical Deep-Dive
* **Non-Technical:** A return receipt signed by the mail carrier upon delivery, filed back into the company’s cabinet.
* **Technical:** Unauthenticated public endpoint exempted from JWT middleware via MPM's `ThirdPartyApiResolverService`. Resolves tenant ID from route parameter and activates tenant database proxy context.

#### 5. NestJS & TypeScript Mental Model
* Like a public webhook controller bypassing JWT AuthGuard with dynamic tenant middleware:
  ```typescript
  @Post('webhook/:companyId')
  @Public() // Bypasses JWT Guard
  async handleWebhook(@Param('companyId') companyId: number, @Body() payload: TwilioWebhookDto) {
    await this.tenantContext.setTenant(companyId);
    await this.logService.updateStatus(payload.MessageSid, payload.MessageStatus);
  }
  ```

#### 6. Files Involved
* **Create:** `src/backend/app/Services/ApiResolvers/WhatsappApiResolver.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Http/Controllers/WhatsappWebhookController.php`
* **Modify:** [`src/backend/app/Services/ApiResolvers/ThirdPartyApiResolverService.php`](file:///home/elie/Documents/OSEAS-Hackathon/Personal/micropowermanager/src/backend/app/Services/ApiResolvers/ThirdPartyApiResolverService.php)
* **Modify:** `src/backend/app/Plugins/Whatsapp/routes/api.php`

#### 7. Example Workflow
1. Twilio delivers message to WhatsApp handset.
2. Twilio fires POST request to `https://{domain}/api/whatsapp/webhook/1`.
3. `ThirdPartyApiResolverService` recognizes `api/whatsapp/` prefix and invokes `WhatsappApiResolver`.
4. Tenant 1 database context is established.
5. Controller marks log row `status = 'delivered'`.

---

### Task 10: Configurable Fallback Policy Engine
**Priority:** P2 (Advanced Standout Feature)  
**Assigned Developer:** Developer 4 (Webhooks & QA) / Developer 2 (Engine)

#### 1. Plain English Summary
Operators get to choose their delivery policy on the settings screen:
* **Immediate Fallback:** If WhatsApp returns an error or unreached status immediately, fire SMS within 5 seconds.
* **Smart Retry Window (5 min):** Send WhatsApp; if delivery receipt isn't received within 5 minutes, automatically fire the SMS fallback.
* **Dual Dispatch:** Send both WhatsApp and SMS concurrently (great for critical onboarding).

#### 2. User Story
> **As an** operator running a remote mini-grid,  
> **I want** to choose between Immediate Fallback and Smart Retry,  
> **So that** I balance SMS cost savings against rapid token delivery.

#### 3. Acceptance Criteria
* [ ] Policy selection stored in `whatsapp_configs.fallback_strategy`.
* [ ] Policy behavior enforced in `WhatsappGateway`:
  * If `immediate`: Failures trigger instant SMS dispatch.
  * If `smart_retry`: Delays fallback via scheduled queue job (`dispatch(new FallbackSmsJob(...))->delay(now()->addMinutes(5))`).
  * If `dual_dispatch`: Dispatches both WhatsApp and SMS simultaneously.

#### 4. Files Involved
* **Modify:** `src/backend/app/Plugins/Whatsapp/WhatsappGateway.php`
* **Create:** `src/backend/app/Plugins/Whatsapp/Jobs/DelayedFallbackSmsJob.php`

---

### Task 11: End-to-End Simulation, Bruno Collection & Docs
**Priority:** P0 (Submission & Presentation)  
**Assigned Developer:** All Team Members

#### 1. Plain English Summary
We verify the entire integration with a real payment simulation, create a complete Bruno API test collection so judges can test every endpoint with one click, and write clear documentation.

#### 2. Acceptance Criteria
* [ ] Bruno collection files added in `collections/Plugins/WhatsApp/`:
  * `Get Config.bru`
  * `Save Config.bru`
  * `Send Test Ping.bru`
  * `Fetch Logs.bru`
  * `Inbound Webhook.bru`
* [ ] End-to-End Verification: Simulating a payment generates a token and delivers a WhatsApp message on a real phone.
* [ ] Forced Failure Verification: Turning off WhatsApp or supplying an invalid token automatically fires an SMS.
* [ ] Documentation written at `docs/development/plugins/whatsapp.md`.
* [ ] Clean pull request opened against upstream repository.

---
