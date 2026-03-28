contiMusic Store Data 2

This folder contains a denormalized CSV export of a music-store transactional database (similar to the Chinook sample model), plus a schema diagram image.

## Contents

- `album.csv`
- `album2.csv`
- `artist.csv`
- `customer.csv`
- `employee.csv`
- `genre.csv`
- `invoice.csv`
- `invoice_line.csv`
- `media_type.csv`
- `playlist.csv`
- `playlist_track.csv`
- `track.csv`
- `schema_diagram.png`

## Dataset Summary

| File                 | Rows | Key columns                                         | Description                                                                           |
| -------------------- | ---: | --------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `album.csv`          |  347 | `album_id`, `artist_id`                             | Album master data (version with punctuation/special chars retained in many titles).   |
| `album2.csv`         |  347 | `album_id`, `artist_id`                             | Alternate album title variant for the same `album_id` range (many normalized titles). |
| `artist.csv`         |  275 | `artist_id`                                         | Artist reference table.                                                               |
| `customer.csv`       |   59 | `customer_id`, `support_rep_id`                     | Customer profile and contact/billing metadata.                                        |
| `employee.csv`       |    9 | `employee_id`, `reports_to`                         | Internal employee hierarchy (sales/support org).                                      |
| `genre.csv`          |   25 | `genre_id`                                          | Track genre lookup.                                                                   |
| `invoice.csv`        |  614 | `invoice_id`, `customer_id`                         | Invoice header with billing info and invoice total.                                   |
| `invoice_line.csv`   | 4757 | `invoice_line_id`, `invoice_id`, `track_id`         | Invoice line items (track-level purchases).                                           |
| `media_type.csv`     |    5 | `media_type_id`                                     | Media format lookup (MPEG, AAC, etc.).                                                |
| `playlist.csv`       |   18 | `playlist_id`                                       | Playlist names.                                                                       |
| `playlist_track.csv` | 8715 | `playlist_id`, `track_id`                           | Many-to-many bridge between playlists and tracks.                                     |
| `track.csv`          | 3503 | `track_id`, `album_id`, `media_type_id`, `genre_id` | Track catalog with duration, file size, and unit price.                               |

## Data Model (Logical Relationships)

Primary relationships implied by key columns:

- `artist.artist_id` 1 -> many `album.artist_id`
- `album.album_id` 1 -> many `track.album_id`
- `media_type.media_type_id` 1 -> many `track.media_type_id`
- `genre.genre_id` 1 -> many `track.genre_id`
- `customer.customer_id` 1 -> many `invoice.customer_id`
- `invoice.invoice_id` 1 -> many `invoice_line.invoice_id`
- `track.track_id` 1 -> many `invoice_line.track_id`
- `playlist.playlist_id` many <-> many `track.track_id` via `playlist_track`
- `employee.employee_id` 1 -> many `customer.support_rep_id`
- `employee.employee_id` 1 -> many `employee.reports_to` (self-referencing org chart)

You can also inspect the visual schema in `schema_diagram.png`.

## Column-Level Notes

### `track.csv`

- `milliseconds` stores track duration in ms.
- `bytes` stores file size.
- `unit_price` appears as decimal currency values.
- `composer` is frequently blank.

### `invoice.csv` and `invoice_line.csv`

- Header/detail pattern:
  - `invoice.total` is invoice-level amount.
  - `invoice_line.unit_price * quantity` gives line-level subtotal.
- Typical analytics require joining header and lines on `invoice_id`.

### `customer.csv`

- Contains personal contact data (email, phone, address).
- Several optional fields are sparse (`company`, `fax`, `state`, etc.).

### `employee.csv`

- Includes `reports_to` and `levels` fields for hierarchy.
- `birthdate` and `hire_date` are in `dd-MM-yyyy HH:mm` style strings.

## Data Quality / Caveats

1. **Character encoding artifacts**

- Some text appears as mojibake (for example names with accented characters), indicating files may not be UTF-8 decoded correctly in some tools.
- If this matters for reporting, open/reload with the correct source encoding or normalize text during ingestion.

2. **Missing values (not exhaustive, but observed at scale)**
   - `customer.csv`: high null rates in `company`, `fax`, and `state`.
   - `invoice.csv`: many missing `billing_state`; some missing `billing_postal_code`.
   - `track.csv`: many missing `composer`.

3. **`album.csv` vs `album2.csv` differences**
   - Both contain 347 rows with the same `album_id` range (`1..347`).
   - 153 album titles differ in punctuation/normalization (example: `Kill 'Em All` vs `Kill Em All`).
   - Pick one as your canonical source before downstream joins to avoid inconsistent title strings.

## Suggested Load Order (for SQL/Data Warehouse)

1. `artist.csv`
2. `genre.csv`
3. `media_type.csv`
4. `employee.csv`
5. `customer.csv`
6. `album.csv` (or `album2.csv`)
7. `track.csv`
8. `playlist.csv`
9. `invoice.csv`
10. `playlist_track.csv`
11. `invoice_line.csv`

## Quick Analysis Starters

### Top customers by spend (SQL)

```sql
SELECT
  c.customer_id,
  c.first_name,
  c.last_name,
  SUM(i.total) AS total_spend
FROM invoice i
JOIN customer c ON c.customer_id = i.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_spend DESC;
```

### Most purchased tracks (SQL)

```sql
SELECT
  t.track_id,
  t.name,
  COUNT(*) AS purchase_count
FROM invoice_line il
JOIN track t ON t.track_id = il.track_id
GROUP BY t.track_id, t.name
ORDER BY purchase_count DESC;
```

### Revenue by genre (SQL)

```sql
SELECT
  g.name AS genre,
  SUM(il.unit_price * il.quantity) AS revenue
FROM invoice_line il
JOIN track t ON t.track_id = il.track_id
JOIN genre g ON g.genre_id = t.genre_id
GROUP BY g.name
ORDER BY revenue DESC;
```

## Practical Recommendation

For most analytics projects:

- Use `album2.csv` if you prefer simplified title strings.
- Use `album.csv` if punctuation fidelity is important.
- Keep all raw files unchanged and apply cleaning/transformation in a separate processing layer.
