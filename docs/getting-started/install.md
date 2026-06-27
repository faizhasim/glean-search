# Installation

## Raycast Store (recommended)

The easiest way to install Glean Search is from the [Raycast Store](https://www.raycast.com/faizhasim/glean-search):

1. Open Raycast
2. Search for "Store"
3. Search for "Glean Search"
4. Press `Enter` to install

After installation, open **Search Glean** to get started.

## Development install

If you want to modify the extension or contribute, install from source:

### Prerequisites

- [Node.js](https://nodejs.org/) version 22 or later
- npm (ships with Node.js)
- [Raycast](https://raycast.com/) with a developer account

### Steps

```bash
# Clone the repository
git clone https://github.com/faizhasim/glean-search.git
cd glean-search

# Install dependencies
npm install

# Link with Raycast
npm run dev
```

The `npm run dev` command opens Raycast and registers the extension from your local checkout. Any changes you make are reflected immediately.

### Building for distribution

```bash
npm run build
```

This produces a production build that can be published to the Raycast Store.
