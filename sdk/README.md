# vektori-js SDK

Official JavaScript SDK for the Vektori analytics platform. Track user behavior, run experiments, and analyze conversion funnels in your web application.

**Current version: 3.2.1**

## Supported Frameworks

- JavaScript (vanilla)
- React
- Vue
- Angular

All major frontend frameworks are supported out of the box with dedicated bindings.

## Installation

```bash
npm install @vektori/js-sdk
# or
yarn add @vektori/js-sdk
```

## Quick Start

```javascript
import { Vektori } from '@vektori/js-sdk';

const vektori = new Vektori({
  apiKey: 'your-api-key',
  environment: 'production',  // or 'staging'
});

// Initialize tracking
vektori.init();
```

## Core API

### Identify Users

Associate events with a specific user. Call this after login or when user identity is known.

```javascript
vektori.identify('user-123', {
  email: 'user@example.com',
  plan: 'pro',
  company: 'Acme Inc',
  signupDate: '2025-06-15',
});
```

### Track Events

Record custom events with optional properties.

```javascript
vektori.track('product_added_to_cart', {
  productId: 'SKU-4521',
  price: 49.99,
  currency: 'EUR',
  category: 'electronics',
});
```

### Track Pageviews

Automatic pageview tracking is enabled by default. For SPAs, you can manually trigger pageviews on route changes:

```javascript
vektori.page({
  path: '/products/electronics',
  title: 'Electronics - MyStore',
  referrer: document.referrer,
});
```

### Funnel Tracking

Define and track conversion funnels:

```javascript
vektori.funnel('checkout_flow', {
  step: 'payment_info',
  stepNumber: 3,
  totalSteps: 4,
});
```

## React Integration

```jsx
import { VektoriProvider, useTrack } from '@vektori/js-sdk/react';

function App() {
  return (
    <VektoriProvider apiKey="your-api-key">
      <MyApp />
    </VektoriProvider>
  );
}

function ProductPage({ product }) {
  const track = useTrack();

  const handleAddToCart = () => {
    track('add_to_cart', { productId: product.id, price: product.price });
  };

  return <button onClick={handleAddToCart}>Add to Cart</button>;
}
```

## Vue Plugin

```javascript
import { createApp } from 'vue';
import { VektoriPlugin } from '@vektori/js-sdk/vue';

const app = createApp(App);
app.use(VektoriPlugin, { apiKey: 'your-api-key' });
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
|  | string | required | Your Vektori API key |
|  | string |  | Environment name |
|  | boolean |  | Track pageviews automatically |
|  | number |  | Events per batch |
|  | number |  | Batch flush interval (ms) |
|  | boolean |  | Enable console logging |

## Event Batching

Events are batched automatically for performance. The SDK queues events and sends them in batches of 10 (configurable) or every 5 seconds, whichever comes first. On page unload, remaining events are flushed via .

## Rate Limits

The SDK respects Vektori API rate limits (100 requests/second per API key). If limits are hit, events are queued locally and retried with exponential backoff.

## Webhook Integration

Events can trigger webhooks configured in the Vektori dashboard. Use the v2 webhook format for all new integrations. See the [API Reference](../docs/api-reference.md) for webhook payload structure.

## Changelog

- **3.2.1** (Feb 2026): Fixed batch flush on SPA route changes
- **3.2.0** (Jan 2026): Added Vue and Angular bindings
- **3.1.0** (Nov 2025): Funnel tracking API
- **3.0.0** (Sep 2025): Complete rewrite, dropped IE11 support

## License

MIT
