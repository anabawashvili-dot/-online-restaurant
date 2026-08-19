# Savora — Online Restaurant (Angular)

Angular + SCSS single-page application for browsing a restaurant menu and
placing an order.

- **Menu page** — browse dishes, filter by category (Salads, Soups,
  Chicken-Dishes, Beef-Dishes, Seafood-Dishes, Vegetable-Dishes, Bits&Bites,
  On-The-Side), by spiciness (0–4), by walnuts, and by vegetarian. Each dish
  card shows an image, name, spice level, walnut/vegetarian tags, price,
  average rating, and an "Add to cart" button.
- **Dish detail page** — full description, ingredients, and a 5-star review
  section with a comment form.
- **Cart page** — lists everything added to the basket, lets the customer
  change the quantity of each item or remove it, and shows a live,
  dynamically calculated total.
- **Checkout** — a short registration step (name + phone) followed by an
  order summary and a "request a call to confirm" action.
- **Account page** — registration/profile management, reachable from the
  header at any time.
- **Chefs page** — team profiles with their own comment section.

Data comes from the IT Step Academy Restaurant API:
`https://restaurant.stepprojects.ge/swagger/index.html`

## Getting started

```bash
npm install
npm start        # ng serve — http://localhost:4200
```

Production build:

```bash
npm run build     # output in dist/restaurant-app
```

## Project structure

```
src/app/
  models/            Dish, DishType, CartItem, FilterState, User, Chef, Review
  directives/
    tilt.directive.ts      Reusable 3D tilt + cursor-shine effect for cards
    magnetic.directive.ts  Subtle cursor-pull effect for primary buttons
  services/
    dish.service.ts    HTTP calls to the API + mapping to the app's Dish model
    cart.service.ts    Basket state (Angular signals) + localStorage persistence
    auth.service.ts    Registration used to gate checkout
    review.service.ts  Star ratings + comments per dish, persisted to localStorage
    chef-comment.service.ts  Comments per chef, persisted to localStorage
    theme.service.ts   Light/dark theme toggle, persisted to localStorage
    mock-data.ts       Local sample dishes, used only if the API is unreachable
  components/
    header/            Top nav — cart badge, account link, theme toggle, burger menu on mobile
    carousel/           Auto-rotating "chef's picks" carousel (top 5 priciest dishes)
    dish-list/          Menu page (carousel + filters + grid of dish cards)
    filters/            Category / spice / walnut / vegetarian filter panel
    dish-card/           Single dish card — links through to its detail page
    dish-detail/         Full dish page: image, description, ingredients, reviews, add to cart
    spice-meter/         Reusable chili gauge (read-only or interactive), 0-4
    star-rating/          Reusable 5-star gauge (read-only average or interactive picker)
    cart/                Basket page
    cart-item/            One basket row (quantity stepper, remove, line total)
    checkout-dialog/     Order summary → "we'll call you" confirmation
    account/              Registration / profile page
    chefs/                Team page with per-chef comment sections
    footer/               Site footer shown on every page
    not-found/             404 page shown for any unmatched route
    scroll-top/            Floating "back to top" button
    cursor-spotlight/       Soft light that follows the cursor across the whole app
environments/
  environment.ts         apiUrl points at the restaurant API
```

## API reference

Base API: `RestaurantAPI` (`https://restaurant.stepprojects.ge/api`).

**Endpoints used:**
- `GET /api/Products/GetAll` — all dishes
- `GET /api/Categories/GetAll` — all categories (`{ id, name }`)

The API also exposes `GET /api/Products/GetFiltered` and a `Baskets`
controller (`GetAll` / `AddToBasket` / `UpdateBasket` /
`DeleteProduct/{id}`); this app keeps the basket client-side in
`CartService` instead, so those endpoints aren't called. To move the basket
server-side, wire them into `CartService`.

**Product shape:**

```json
{
  "id": 0,
  "name": "string",
  "price": 0,
  "nuts": true,
  "image": "string",
  "vegeterian": true,
  "spiciness": 0,
  "categoryId": 0
}
```

Notes on the schema:

- `nuts` reflects whether a dish contains walnuts (a common ingredient in
  Georgian dishes such as bazhe and satsivi) — not garlic.
- `vegeterian` is spelled without the second "a" in the API response;
  `DishService.mapDish()` reads that exact key.
- There's no `description` or `ingredients` field on the Products endpoint.
  `Dish.description` / `Dish.ingredients` are simply empty for live data;
  the UI hides those sections when empty.
- `categoryName` isn't included on the product — only `categoryId`.
  `DishService.getDishes()` fetches both endpoints and resolves each
  product's category name locally.

### Dish photos

`DishService.resolveImageUrl()` resolves relative image paths (e.g.
`/images/soup.jpg`) against the API's site origin, since static files are
typically served from the site root rather than under `/api`. Any dish
whose image fails to load falls back to a placeholder icon automatically
(`onImageError()` in `dish-card`, `cart-item`, `dish-detail`, and
`carousel`).

If the API is unreachable, the app falls back to 16 sample dishes in
`src/app/services/mock-data.ts` (covering every category, several spice
levels, and walnut/vegetarian combinations), so the UI stays demoable
offline.

## Reviews & star ratings

Each dish has a 5-star rating and a comment section on its detail page:

- Anyone can leave a rating, name, and comment.
- The dish card and dish detail page show the average rating and review
  count once a dish has at least one review.
- Reviews persist in `localStorage` (`ReviewService`, key
  `restaurant-reviews`) — the API doesn't expose a ratings endpoint.
- Submitting a review on a dish also posts a linked comment to the chef
  responsible for that dish's category (see `getChefForCategory()` in
  `chefs-data.ts`), so chef profiles reflect feedback on the dishes they
  cook.

## Chefs page (`/chefs`)

Six chef profiles (name, role, specialty, bio) in a card grid, each with its
own expandable comment section (rating + name + comment). Reachable from
the header, footer, and mobile menu. Content lives in
`src/app/services/chefs-data.ts`.

## Checkout flow

Clicking **Checkout** on the basket page opens a dialog:

1. If not yet registered, it asks for name + phone (the same account used
   on the `/account` page).
2. **Confirm** — shows the saved contact details (editable) and an order
   summary with the live total, then a **"Request a call to confirm"**
   button.
3. **Success** — the basket is cleared and the customer sees a confirmation
   that the restaurant will call them.

This is a "request a call" pattern rather than online payment, so there's
no payment integration involved. To submit a real order to a backend, add
an `HttpClient` call inside `CheckoutDialogComponent.requestCall()`.

## Account page (`/account`)

Reachable from the header at any time. First visit shows a registration
form (name and phone required; email and delivery address optional). Once
submitted, the page becomes a profile view with "Edit details" and "Log
out" actions. The header nav shows the customer's name once registered.

## Light / dark theme

The header's 🌙/☀️ button toggles between a bright cream theme and a dark
theme, with no page reload. Both palettes are defined as CSS custom
properties in `styles.scss` (`:root` for light, `:root[data-theme='dark']`
for dark); `ThemeService` sets the `data-theme` attribute on `<html>` and
persists the choice in `localStorage`.

## Responsive design

- **Header** — below 760px the horizontal nav is replaced by a burger
  button that opens a mobile menu, plus a cart-shortcut icon.
- **Menu page** — the filters sidebar moves above the dish grid below
  900px; the grid itself reflows its column count at any width.
- **Carousel, cart rows, dish detail, footer** — each reflows into a single
  column on narrow screens.
- **Checkout dialog** — becomes a bottom sheet below 420px.

## Animation

- Dish cards, chef cards, and the loading skeleton fade/slide in with a
  staggered delay per grid position.
- Dish cards and chef cards tilt in 3D toward the cursor with a light
  "shine" that tracks the pointer (`appTilt` directive), skipped for touch
  input and `prefers-reduced-motion`.
- Registration and review forms show a checkmark next to each field once
  it's valid; successful registration shows a brief checkmark-and-confetti
  confirmation.
- The category chip row on the menu page mirrors the sidebar category
  filter — clicking a chip toggles it in both places.
- The basket total counts up/down smoothly whenever it changes, instead of
  jumping straight to the new number.
- A "back to top" button fades in once the page is scrolled down.
- A soft light follows the cursor across the whole app (`CursorSpotlightComponent`),
  and primary buttons (Add to cart, Checkout, Continue, Request a call) pull
  gently toward the pointer as it gets close (`appMagnetic` directive).
  Both are desktop-mouse only.
- Every route change fades the page content in instead of popping in
  instantly.
- All animations respect `prefers-reduced-motion`.

## Design

Cream/white theme by default (paprika and mustard accents), with a dark
theme available via the header toggle. Fraunces for display type, Inter for
UI text, IBM Plex Mono for prices and counters.
