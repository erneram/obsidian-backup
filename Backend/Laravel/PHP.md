## Switch between versions:
> ⚠️ if version is already installed, skip step 1 & 2
1. First if its not already tapped
```terminal
brew tap shivammathur/php
```
2. Then install the version needed
```terminal
brew install shivammathur/php/php@{version}
```
3. Unlink current version and set the default version used on terminal 
```terminal
brew unlink php@{previous_version}
brew link --overwrite --force php@{version}
```
4. Verify installation
```terminal
php -v
```

---
## Use Snippet for `~/.zshrc`
1. Paste snippet in `~/.zshrc`
```sh
switch-php() {
  if [ -z "$1" ]; then
    echo "⚠️ Usage: switch-php {version}"
    echo "Example: switch-php 8.2"
    return 1
  fi

  version=$1
  formula="shivammathur/php/php@${version}"

  # Check if the version is installed
  if ! brew list --versions php@${version} >/dev/null 2>&1; then
    echo "📦 PHP ${version} is not installed. Installing..."
    brew install $formula
  fi

  # Unlink all known versions
  echo "🔗 Unlinking other versions..."
  for v in 7.4 8.0 8.1 8.2 8.3 8.4; do
    brew unlink php@$v >/dev/null 2>&1
  done
  brew unlink php >/dev/null 2>&1

  # Link the requested version
  echo "➡️ Linking php@${version}..."
  brew link --overwrite --force php@${version}

  # Update PATH dynamically (only for this session)
  export PATH="/opt/homebrew/opt/php@${version}/bin:$PATH"
  export PATH="/opt/homebrew/opt/php@${version}/sbin:$PATH"

  # Show the active version
  echo "✅ Active PHP version:"
  php -v | head -n 1
}
```
1. Reload shell config
```sh
source ~/.zshrc
```
3. Switch version using
```sh
switch-php 8.2
switch-php 8.3
switch-php 7.4
```