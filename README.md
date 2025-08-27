Redis converts Geospatial data (latitude and longitude) into a single "score" value so that it can be stored in a [Sorted Set](https://redis.io/docs/latest/develop/data-types/sorted-sets/).

This repository explains the algorithm used for this conversion.

# Encoding

Encoding is a 3-step process.

1. Latitude and longitude are normalized to the [0, 2^26) range
2. The normalized values are truncated to integers (i.e. the decimal part is dropped)
3. The bits of the normalized latitude and longitude values are interleaved to get a 52-bit integer value

## Step 1: Normalization

The values of latitude and longitude are normalized to values in the [0, 2^26) range.

The official Redis source code implements this in [geohash.c](https://github.com/redis/redis/blob/ff2f0b092c24d5cc590ff1eb596fc0865e0fb721/src/geohash.c#L141-L148).

Here's some pseudocode illustrating how this is done:

```python
MIN_LATITUDE = -85.05112878
MAX_LATITUDE = 85.05112878
MIN_LONGITUDE = -180
MAX_LONGITUDE = 180

LATITUDE_RANGE = MAX_LATITUDE - MIN_LATITUDE
LONGITUDE_RANGE = MAX_LONGITUDE - MIN_LONGITUDE

normalized_latitude = 2^26 * (latitude - MIN_LATITUDE) / LATITUDE_RANGE
normalized_longitude = 2^26 * (longitude - MIN_LONGITUDE) / LONGITUDE_RANGE
```

> [!NOTE]
> The latitude range Redis accepts is +/-85.05° and not +/-90°. This is because of the [Web Mercator projection](https://en.wikipedia.org/wiki/Web_Mercator_projection) used to project the Earth onto a 2D plane.

These normalized values are combined in the next steps to calculate a "score".

## Step 2: Truncation

The normalized values (floats) are truncated to integers. This is not rounding, the decimal part is dropped entirely.

In the Redis source code, this conversion happens implicitly when the `double` values are casted to `uint32_t` [here](https://github.com/redis/redis/blob/ff2f0b092c24d5cc590ff1eb596fc0865e0fb721/src/geohash.c#L149).

In Python, this conversion can be done explicitly using the `int()` function:

```python
normalized_latitude = int(normalized_latitude)
normalized_longitude = int(normalized_longitude)
```

## Step 3: Interleaving

The bits of the normalized latitude and longitude values are then interleaved to get a 64-bit integer value.

In the Redis source code, this is done in the [`interleave64` function](https://github.com/redis/redis/blob/eac48279ad21b8612038953fefa0dcf926773efc/src/geohash.c#L52-L77).

Here's some pseudocode illustrating how this is done:

```python
def interleave(x: int, y: int) -> int:
    # First, the values are spread from 32-bit to 64-bit integers.
    # This is done by inserting 32 zero bits in-between.
    #
    # Before spread: x1  x2  ...  x31  x32
    # After spread:  0   x1  ...   0   x16  ... 0  x31  0  x32
    x = spread_int32_to_int64(x)
    y = spread_int32_to_int64(y)

    # The y value is then shifted 1 bit to the left.
    # Before shift: 0   y1   0   y2 ... 0   y31   0   y32
    # After shift:  y1   0   y2 ... 0   y31   0   y32   0
    y_shifted = y << 1

    # Next, x and y_shifted are combined using a bitwise OR.
    #
    # Before bitwise OR (x): 0   x1   0   x2   ...  0   x31    0   x32
    # Before bitwise OR (y): y1  0    y2  0    ...  y31  0    y32   0
    # After bitwise OR     : y1  x2   y2  x2   ...  y31  x31  y32  x32
    return x | y_shifted

# Spreads a 32-bit integer to a 64-bit integer by inserting
# 32 zero bits in-between.
#
# Before spread: x1  x2  ...  x31  x32
# After spread:  0   x1  ...   0   x16  ... 0  x31  0  x32
def spread_int32_to_int64(v: int) -> int:
    # Ensure only lower 32 bits are non-zero.
    v = v & 0xFFFFFFFF

    # Bitwise operations to spread 32 bits into 64 bits with zeros in-between
    v = (v | (v << 16)) & 0x0000FFFF0000FFFF
    v = (v | (v << 8))  & 0x00FF00FF00FF00FF
    v = (v | (v << 4))  & 0x0F0F0F0F0F0F0F0F
    v = (v | (v << 2))  & 0x3333333333333333
    v = (v | (v << 1))  & 0x5555555555555555

    return v

score = interleave(normalized_latitude, normalized_longitude)
```

Here's a visual description of the spread algorithm:
![spread](./media/Spread.gif)


# Decoding

Decoding a score converts it back to the original latitude and longitude values. This is essentially the reverse of the encoding process.

## Step 1: Separating the Interleaved Bits

First, we need to separate the interleaved latitude and longitude bits from the score. Since longitude bits were shifted left by 1 during encoding, we need to shift them back.

```python
# Extract longitude bits (they were shifted left by 1 during encoding)
y = geo_code >> 1

# Extract latitude bits (they were in the original positions)
x = geo_code
```

## Step 2: Compacting 64-bit integer to 32-bit integers

The bits were spread from 32-bit to 64-bit integers during encoding. Now we need to compact them back to 32-bit integers.

```python
# Compact both latitude and longitude back to 32-bit integers
grid_latitude_number = compact_int64_to_int32(x)
grid_longitude_number = compact_int64_to_int32(y)
```

Here's a pseudocode illustrating how int64 is compacted to int32.
```python
def compact_int64_to_int32(v: int) -> int:
    # Keep only the bits in even positions
    v = v & 0x5555555555555555

    # Before masking: w1   v1  ...   w2   v16  ... w31  v31  w32  v32
    # After masking: 0   v1  ...   0   v16  ... 0  v31  0  v32

    # Where w1, w2,..w31 are the digits from longitude if we're compacting latitude, or digits from latitude if we're compacting longitude
    # So, we mask them out and only keep the relevant bits that we wish to compact

    # ------
    # Reverse the spreading process by shifting and masking
    v = (v | (v >> 1)) & 0x3333333333333333
    v = (v | (v >> 2)) & 0x0F0F0F0F0F0F0F0F
    v = (v | (v >> 4)) & 0x00FF00FF00FF00FF
    v = (v | (v >> 8)) & 0x0000FFFF0000FFFF
    v = (v | (v >> 16)) & 0x00000000FFFFFFFF

    # Before compacting: 0   v1  ...   0   v16  ... 0  v31  0  v32
    # After compacting: v1  v2  ...  v31  v32
    # -----
    
    return v
```

Here's a visual description of the compaction algorithm:
![compaction](./media/Compact.gif)

## Step 3: Converting Back to Geographic Coordinates

The decoded 32-bit integers represent grid cell numbers. We convert them back to geographic coordinates by reversing the normalization process. Here's a pseudocode illustrating this process.

```python
def convert_grid_numbers_to_coordinates(grid_latitude_number, grid_longitude_number) -> (float, float):
    # Calculate the grid boundaries
    grid_latitude_min = MIN_LATITUDE + LATITUDE_RANGE * (grid_latitude_number / (2**26))
    grid_latitude_max = MIN_LATITUDE + LATITUDE_RANGE * ((grid_latitude_number + 1) / (2**26))
    grid_longitude_min = MIN_LONGITUDE + LONGITUDE_RANGE * (grid_longitude_number / (2**26))
    grid_longitude_max = MIN_LONGITUDE + LONGITUDE_RANGE * ((grid_longitude_number + 1) / (2**26))
    
    # Calculate the center point of the grid cell
    latitude = (grid_latitude_min + grid_latitude_max) / 2
    longitude = (grid_longitude_min + grid_longitude_max) / 2
    return (latitude, longitude)
```

> [!NOTE]
> The decoded coordinates represent the center of a grid cell, not the exact original coordinates. This is because the encoding process truncates coordinates to grid cells. The precision depends on the grid resolution (which is determined by the 26-bit normalization).

# FAQ

### Why is the latitude range from -85.05° to 85.05° and not -90° to 90°?

This is because of the [Web Mercator projection](https://en.wikipedia.org/wiki/Web_Mercator_projection) used to project the Earth onto a 2D plane.

### Why are the latitude and longitude normalized to 26-bit integers instead of 32-bit integers?

In [sorted sets](https://redis.io/docs/latest/data-types/sorted-sets/), scores are stored as [float64](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) numbers. Float64 values have limits on integer precision. They [can represent integers upto 2^53 accurately](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#Precision_limitations_on_integer_values) but start to lose precision after that. Using 26-bit integers ensures that the final score is under 2^52 and thus within the integer precision limits of float64.

### How do I test my implementation?

Here are some example values to test your implementation against:

| Place      | Latitude | Longitude | Score              |
| ---------- | -------- | --------- | -------------------|
| Bangkok    | 13.7220  | 100.5252  | 3962257306574459.0 |
| Beijing    | 39.9075  | 116.3972  | 4069885364908765.0 |
| Berlin     | 52.5244  | 13.4105   | 3673983964876493.0 |
| Copenhagen | 55.6759  | 12.5655   | 3685973395504349.0 |
| New Delhi  | 28.6667  | 77.2167   | 3631527070936756.0 |
| Kathmandu  | 27.7017  | 85.3206   | 3639507404773204.0 |
| London     | 51.5074  | -0.1278   | 2163557714755072.0 |
| New York   | 40.7128  | -74.0060  | 1791873974549446.0 |
| Paris      | 48.8534  | 2.3488    | 3663832752681684.0 |
| Sydney     | -33.8688 | 151.2093  | 3252046221964352.0 |
| Tokyo      | 35.6895  | 139.6917  | 4171231230197045.0 |
| Vienna     | 48.2064  | 16.3707   | 3673109836391743.0 |
