# Add Optional Dark Mode

## Description

This PR adds an optional dark mode feature to TeslaMate, allowing users to choose between light mode, dark mode, or following their system preference.

## Motivation

Dark mode has become a standard feature in modern web applications, reducing eye strain in low-light conditions and providing a more comfortable viewing experience for many users. This implementation maintains TeslaMate's clean aesthetic while offering users the flexibility to choose their preferred theme.

## Screenshots

### Light Mode (Default)

Current appearance - no changes for existing users

### Dark Mode

![Dark Mode Example](https://via.placeholder.com/800x400?text=Please+add+screenshot+of+dark+mode)

### Settings Page

![Theme Settings](https://via.placeholder.com/800x400?text=Please+add+screenshot+of+theme+settings)

## Features

### Three Appearance Options

- **Light** (default): Always use light theme
- **Follow System**: Automatically match OS/browser dark mode preference via `prefers-color-scheme`
- **Dark**: Always use dark theme

### User Experience

- ✅ Theme preference stored in database
- ✅ Instant theme switching without page reload
- ✅ No flash of wrong theme on initial page load
- ✅ Comprehensive styling for all UI components
- ✅ Monochrome map tiles in dark mode (Tesla-inspired aesthetic)
- ✅ System preference monitoring (updates automatically when OS theme changes)

### Backward Compatibility

- ✅ Defaults to light mode for existing users
- ✅ Non-breaking database migration
- ✅ No changes to existing light mode appearance

## Technical Implementation

### Database Changes

- New `theme_mode` field in `settings` table
- Enum type with values: `light`, `system`, `dark`
- Default value: `light` (maintains current behavior)
- Migration: `20251207212310_add_theme_mode_to_settings.exs`

### Frontend Changes

1. **Dark Mode Styles** (`assets/css/dark-mode.scss`)

   - CSS custom properties for theme colors
   - Comprehensive component styling (cards, tables, forms, modals, etc.)
   - Inverted map tiles with grayscale filter for Tesla-like monochrome appearance

2. **Theme Switching Logic** (`assets/js/main.js`)

   - Detects system color scheme preference
   - Applies theme without flash on page load
   - Monitors system preference changes

3. **LiveView Hook** (`assets/js/hooks.js`)

   - `ThemeSelector` hook for instant theme changes
   - Updates theme immediately when changed in settings

4. **Settings UI** (`lib/teslamate_web/live/settings_live/index.html.heex`)

   - New "Theme" section with dropdown selector
   - Positioned logically with other UI preferences

5. **Layout Template** (`lib/teslamate_web/templates/layout/root.html.heex`)
   - Theme data attributes on HTML element
   - Inline script to prevent theme flash

### Backend Changes

- Updated `GlobalSettings` schema to include `theme_mode` field
- Updated changeset to validate theme preference

## Testing Performed

- [x] Theme changes apply instantly in settings
- [x] Theme persists across page navigation
- [x] "Follow System" mode respects OS preference
- [x] Maps render correctly in both modes
- [x] All UI components styled appropriately in dark mode
- [x] No flash of wrong theme on page load
- [x] Migration runs successfully
- [x] Existing installations default to light mode

## Files Changed

- `assets/css/app.scss` - Import dark mode styles
- `assets/css/dark-mode.scss` - Dark theme implementation (new)
- `assets/js/hooks.js` - ThemeSelector hook and dark mode map tiles
- `assets/js/main.js` - Theme switching logic
- `lib/teslamate/settings/global_settings.ex` - Add theme_mode field
- `lib/teslamate_web/live/settings_live/index.html.heex` - Theme settings UI
- `lib/teslamate_web/templates/layout/root.html.heex` - Theme initialization
- `priv/repo/migrations/20251207212310_add_theme_mode_to_settings.exs` - Database migration (new)

## Checklist

- [x] Code follows the project's coding standards
- [x] Changes are backward compatible
- [x] Database migration included and tested
- [x] No breaking changes for existing users
- [x] Feature is opt-in (defaults to current behavior)
- [x] All UI components tested in both themes
- [x] Documentation updated (if needed)

## Additional Notes

- The dark mode map tiles use a monochrome style inspired by Tesla's in-car navigation, providing a clean, modern look
- The implementation uses CSS filters to invert standard OSM tiles rather than loading third-party dark tiles, reducing external dependencies
- Theme changes are instant and don't require page reloads, providing a smooth user experience
