on:
  push:
    branches:
      - main

name: CI — Validate, Accessibility check, and Deploy to GitHub Pages

jobs:
  test:
    name: Validate + Accessibility (pa11y)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install validation and test tools
        run: |
          npm install -g html-validator@7 http-server pa11y

      - name: HTML validation (index.html)
        # Validate the main page; modify to validate other files if present
        run: |
          echo "Running HTML validation on index.html..."
          # html-validator prints errors and returns non-zero on issues
          npx html-validator --file=index.html --format=text

      - name: Start a simple HTTP server for accessibility testing
        run: |
          echo "Starting http-server on port 8080..."
          npx http-server . -p 8080 --silent &>/dev/null &
          sleep 2

      - name: Accessibility test with Pa11y
        run: |
          echo "Running pa11y against the local server..."
          npx pa11y http://127.0.0.1:8080/ -r cli || true
        # We intentionally do not fail the job on pa11y warnings (change || true to remove that behavior)

  deploy:
    name: Deploy to GitHub Pages
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Prepare publish directory
        run: |
          # Copy only the static site files to a clean publish directory.
          mkdir -p public
          rsync -av --delete \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='node_modules' \
            --exclude='public' \
            ./ public/ >/dev/null

      - name: Show publish dir contents
        run: |
          echo "Files to be published:"
          ls -la public

      - name: Deploy to gh-pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: gh-pages
          keep_files: false
