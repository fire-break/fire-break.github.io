---
title: "How to build a DoH service in Cloudflare Workers"
layout: default 
---

## **How to build a DoH service in Cloudflare Workers**

Here's how to build a DoH service using Cloudflare Workers via the **Web Dashboard** (no command line needed):

## Step 1: Sign Up for Cloudflare

1. Go to [cloudflare.com](https://cloudflare.com)
2. Create a free account
3. Log in to your dashboard

## Step 2: Create a Worker

1. In the Cloudflare dashboard, click **Workers & Pages** in the left sidebar
2. Click **Create application**
3. Click **Create Worker**
4. Give it a name (e.g., `my-doh-service`)
5. Click **Deploy** (you'll edit the code next)

## Step 3: Edit the Worker Code

1. After deployment, click **Edit code** button
2. Delete all the default code
3. Paste the following DoH code:

```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Show info page at root
    if (url.pathname === '/') {
      return new Response(`
        <h1>DoH Service</h1>
        <p>DNS-over-HTTPS endpoint:</p>
        <code>https://${url.host}/dns-query</code>
        <p>Use this URL in your browser or DNS client.</p>
      `, {
        headers: { 'Content-Type': 'text/html' }
      });
    }
    
    // Handle DoH requests
    if (url.pathname !== '/dns-query') {
      return new Response('Not found', { status: 404 });
    }
    
    // Only accept GET and POST requests
    if (!['GET', 'POST'].includes(request.method)) {
      return new Response('Method not allowed', { status: 405 });
    }

    try {
      let dnsQuery;

      if (request.method === 'GET') {
        // Handle GET request (dns parameter is base64url encoded)
        const dnsParam = url.searchParams.get('dns');
        
        if (!dnsParam) {
          return new Response('Missing dns parameter', { status: 400 });
        }

        // Base64url decode
        dnsQuery = base64UrlDecode(dnsParam);
      } else {
        // Handle POST request
        const contentType = request.headers.get('content-type');
        if (contentType !== 'application/dns-message') {
          return new Response('Invalid content type', { status: 400 });
        }

        dnsQuery = await request.arrayBuffer();
      }

      // Forward to upstream DoH server (using Cloudflare's 1.1.1.1)
      const upstreamResponse = await fetch('https://1.1.1.1/dns-query', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/dns-message',
          'Accept': 'application/dns-message',
        },
        body: dnsQuery,
      });

      // Return DNS response
      const dnsResponse = await upstreamResponse.arrayBuffer();
      
      return new Response(dnsResponse, {
        status: 200,
        headers: {
          'Content-Type': 'application/dns-message',
          'Cache-Control': 'max-age=60',
          'Access-Control-Allow-Origin': '*', // Enable CORS if needed
        },
      });

    } catch (error) {
      console.error('DoH error:', error);
      return new Response('Internal server error', { status: 500 });
    }
  },
};

// Base64url decode function
function base64UrlDecode(str) {
  // Convert base64url to standard base64
  str = str.replace(/-/g, '+').replace(/_/g, '/');
  
  // Add padding
  while (str.length % 4) {
    str += '=';
  }
  
  // Decode
  const binaryString = atob(str);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  
  return bytes.buffer;
}
```

4. Click **Save and Deploy**

## Step 4: Get Your DoH URL

After deployment, you'll see your Worker URL, something like:
```
https://my-doh-service.your-subdomain.workers.dev
```

Your **DoH URL** will be:
```
https://my-doh-service.your-subdomain.workers.dev/dns-query
```

## Step 5: Test Your DoH Service

**Option 1: Visit in Browser**

Open your Worker URL (without `/dns-query`) to see the info page:
```
https://my-doh-service.your-subdomain.workers.dev/
```

**Option 2: Use Online DoH Tester**

Visit [DNS Leak Test](https://dnsleaktest.com/) or similar tools to verify.

## Step 6: Configure Your Browser

### Firefox
1. Open Settings → Privacy & Security
2. Scroll to "DNS over HTTPS"
3. Click "Manage Exceptions" or use Custom
4. Enter your DoH URL:
   ```
   https://my-doh-service.your-subdomain.workers.dev/dns-query
   ```

### Chrome/Edge
1. Open Settings → Privacy and security → Security
2. Scroll to "Use secure DNS"
3. Select "With Custom" 
4. Enter your DoH URL

### iOS/macOS
1. Download a DoH profile or use apps like DNSecure
2. Configure with your DoH URL

## Optional: Add Custom Domain

If you have a domain on Cloudflare:

1. Go to your Worker settings
2. Click **Triggers** tab
3. Click **Add Custom Domain**
4. Enter subdomain (e.g., `doh.yourdomain.com`)
5. Click **Add Custom Domain**

Then your DoH URL becomes:
```
https://doh.yourdomain.com/dns-query
```

## Free Tier Limits

Cloudflare Workers Free plan includes:
- **100,000 requests per day**
- Sufficient for personal use
- Automatic global distribution

## Advanced Options (via Web UI)

**Add Caching:**
The code already includes basic caching with `Cache-Control: max-age=60`

**Change Upstream DNS:**
In the code, replace `https://1.1.1.1/dns-query` with:
- `https://dns.google/dns-query` (Google)
- `https://dns.quad9.net/dns-query` (Quad9)
- `https://doh.opendns.com/dns-query` (OpenDNS)

**Monitor Usage:**
1. Go to Workers & Pages
2. Click your Worker
3. View **Metrics** tab for request statistics

That's it! You now have a working DoH service without touching the command line. Your DoH URL is ready to use: `https://your-worker.workers.dev/dns-query` ✅
