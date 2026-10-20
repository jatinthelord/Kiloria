# Image decoding and texture upload

`assets::ImageDecoder` converts bounded BMP and TGA files to
`render::TextureData`, a top-to-bottom RGBA8 buffer accepted directly by
`OpenGLRenderer::createTexture`:

```cpp
const auto resolved = registry.resolve(*registry.find(textureAssetId));
const auto fileBytes = readBoundedFile(resolved); // application/platform I/O
const auto decoded = kiloria::assets::ImageDecoder::decode(fileBytes);
if (!decoded) {
    // Surface decoded.message in the asset browser or loading report.
} else {
    const auto handle = renderer.createTexture(decoded.image);
}
```

The caller owns file access and should resolve project assets through
`assets::Registry`. Automatic format detection recognizes BMP by its signature;
other input is interpreted as TGA. Explicit `ImageFormat::Bitmap` and
`ImageFormat::Targa` selection is available when the source format is known.

Supported formats are uncompressed 24/32-bit BI_RGB BMP, uncompressed 24/32-bit
true-color TGA, 8-bit grayscale TGA, and type-10 true-color TGA RLE. BMP row
padding and top-down images are handled. TGA right/bottom origins and run/raw
packets are normalized to top-to-bottom RGBA. BMP's fourth BI_RGB byte is
treated as padding; its output alpha is opaque. TGA 32-bit alpha is retained.

`ImageDecodeLimits` caps source bytes, dimensions, and total pixels before
allocation; malformed/truncated headers, rows, and RLE packets return structured
errors. PNG, JPEG, palettes, compressed BMP, TGA color maps, mipmap generation,
streaming, and GPU caching are not implemented yet. `kiloria_image_decode`
shows a tiny in-memory TGA decoded without a graphics context.
