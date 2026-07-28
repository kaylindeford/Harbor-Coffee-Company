// api/create-checkout.js
//
// This file goes in a folder called "api" at the root of your project,
// deployed on Vercel (https://vercel.com — free tier is fine).
//
// WHAT THIS DOES
// Your website sends this function the customer's cart + contact info.
// This function (running privately on Vercel's servers, never in the
// browser) calls Square's Checkout API using your SECRET Square access
// token to create a real, itemized Square-hosted checkout page, and
// sends the URL back to your website. Your website then redirects the
// customer's browser to that squareup.com page to actually pay.
//
// Your Square access token NEVER appears in your website's code. It only
// ever lives on Vercel, as an "environment variable" (set up instructions
// below). This is the safe way to do this — never paste your Square
// access token into the HTML file itself.
//
// ---------------------------------------------------------------------
// SETUP STEPS (one-time)
// ---------------------------------------------------------------------
// 1. Create a free account at https://vercel.com (sign up with GitHub is
//    easiest).
// 2. Create a Square application + get your access token:
//    - Go to https://developer.squareup.com/apps
//    - Create an application (any name, e.g. "Harbor Coffee Website")
//    - Under that app, get your PRODUCTION "Access Token" (not sandbox,
//      once you're ready for real payments) and your Location ID
//      (Square Dashboard > your business name > Locations).
// 3. In your project on Vercel, go to Settings > Environment Variables
//    and add:
//      SQUARE_ACCESS_TOKEN  = (your Square access token)
//      SQUARE_LOCATION_ID   = (your Square location ID)
// 4. Put this file at:  /api/create-checkout.js  in the same project
//    you deploy your harbor-coffee.html site from (rename the HTML file
//    to index.html so it's the homepage of that project).
// 5. Deploy. Your website's fetch('/api/create-checkout') calls will now
//    reach this function automatically — no extra wiring needed.
//
// A developer or Kaylin herself can do this; no ongoing coding is
// needed once it's deployed. If anything about your Square account
// changes (new location, regenerated token), just update the two
// environment variables above — the code itself doesn't change.
// ---------------------------------------------------------------------

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    res.status(405).json({ error: 'Method not allowed' });
    return;
  }

  const { items, customer, fulfillment } = req.body || {};

  if (!Array.isArray(items) || items.length === 0) {
    res.status(400).json({ error: 'No items in order' });
    return;
  }

  // Build Square line items from the cart sent by the website.
  // Square line item names have a length limit, so we trim long ones.
  const lineItems = items.map((item) => ({
    name: String(item.name || 'Item').slice(0, 500),
    quantity: String(item.quantity || 1),
    base_price_money: {
      amount: Math.round(item.unitPriceCents || 0),
      currency: 'USD',
    },
  }));

  // A short note on the order so you can see fulfillment details in
  // your Square dashboard alongside the payment.
  const noteParts = [];
  if (fulfillment) {
    if (fulfillment.type === 'subscription') {
      noteParts.push(`Subscription (${fulfillment.frequency || 'weekly'})`);
    } else {
      noteParts.push('Pickup order');
    }
    if (fulfillment.date) noteParts.push(`Date: ${fulfillment.date}`);
    if (fulfillment.time) noteParts.push(`Time: ${fulfillment.time}`);
  }
  if (customer && customer.phone) noteParts.push(`Phone: ${customer.phone}`);
  const orderNote = noteParts.join(' | ').slice(0, 500);

  try {
    const squareRes = await fetch(
      'https://connect.squareup.com/v2/online-checkout/payment-links',
      {
        method: 'POST',
        headers: {
          'Square-Version': '2026-05-20',
          Authorization: `Bearer ${process.env.SQUARE_ACCESS_TOKEN}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          idempotency_key: crypto.randomUUID(),
          order: {
            location_id: process.env.SQUARE_LOCATION_ID,
            line_items: lineItems,
            note: orderNote || undefined,
          },
          checkout_options: {
            // Change this to your real domain once your site is live.
            redirect_url: 'https://your-harbor-coffee-domain.com/',
          },
          pre_populated_data:
            customer && customer.email
              ? { buyer_email: customer.email }
              : undefined,
        }),
      }
    );

    const data = await squareRes.json();

    if (!squareRes.ok) {
      console.error('Square API error:', data);
      res.status(500).json({ error: 'Square could not create the checkout link' });
      return;
    }

    res.status(200).json({ url: data.payment_link.url });
  } catch (err) {
    console.error('Server error creating checkout:', err);
    res.status(500).json({ error: 'Server error creating checkout' });
  }
}
