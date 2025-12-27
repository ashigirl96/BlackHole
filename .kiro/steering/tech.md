# tech

## Development Build Settings
**When**: Building BlackHole locally for development and testing

- Use `CODE_SIGNING_REQUIRED=NO` to bypass code signing during development
- Distribution installers require proper code signing with Developer ID
- Local development and testing work without code signing
- Ignore `MACOSX_DEPLOYMENT_TARGET=10.10` warnings from Xcode
- Current SDK supports 10.13+ but production builds are unaffected by this warning

```bash
# ✅ Good - Development build without signing
xcodebuild -project BlackHole.xcodeproj \
  -configuration Debug \
  -target BlackHole \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO

# ❌ Bad - Will fail without Developer ID certificate
xcodebuild -project BlackHole.xcodeproj \
  -configuration Debug \
  -target BlackHole
```

