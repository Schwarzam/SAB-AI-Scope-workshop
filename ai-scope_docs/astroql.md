# astroql Query Language
route: astroql

# astroql — Lightweight Spatial Query Primitives for Astronomy

`astroql` provides a minimal set of high-level spatial functions designed for astronomical databases.
Instead of writing backend-specific sql (q3c, pgsphere), users express intent with simple primitives:

```sql
cone(…)
dist(…)
```

astroql then rewrites these into efficient sql based on the spatial indexes of each table.

---

## column pattern matching

astroql supports automatic column expansion using wildcard patterns.  
you write short patterns, astroql expands them to actual column names before compiling the query.

**supported patterns**
- `prefix*` — columns starting with `prefix`
- `*suffix` — columns ending with `suffix`
- `*middle*` — columns containing `middle` anywhere

**example explanation**  
If `dr3.vac_photoz` contains columns `mag_pstotal_u`, `mag_pstotal_g`, `mag_pstotal_r`,  
then writing:

```sql
select mag_pstotal*
from dr3.vac_photoz;
```

automatically expands to:

```sql
select mag_pstotal_u, mag_pstotal_g, mag_pstotal_r
from dr3.vac_photoz;
```

astroql also supports alias templates when expanding column wildcards.
This allows you to assign patterned output names automatically.

You can attach an alias containing *, and astroql will replace the * with the actual column name being expanded.

example

```sql
select t.mag_pstotal* as dr3_*
from dr3.vac_photoz t;
```

Assuming the table contains:

```sql
mag_pstotal_u
mag_pstotal_g
mag_pstotal_r
```

astroql expands automatically to:

```sql
select
    t.mag_pstotal_u as dr3_mag_pstotal_u,
    t.mag_pstotal_g as dr3_mag_pstotal_g,
    t.mag_pstotal_r as dr3_mag_pstotal_r
from dr3.vac_photoz t;
```
---

## cone() — cone search

The `cone()` function selects objects inside a circular sky region.

```sql
cone(ra_col, dec_col, center_ra, center_dec, radius_deg)
```

### example 1 — simple cone search  
**what it does:**  
Returns all objects from `dr3.all_dr3` that lie within 0.1 degrees of (RA=180, DEC=0).

```sql
select *
from dr3.all_dr3
where cone(ra, dec, 180.0, 0.0, 0.1);
```

Internally, astroql rewrites this into optimized `q3c_radial_query(...)` or pgSphere queries.

---

### example 2 — matching tables (idr6 ↔ upload)  
**what it does:**  
For each object in your uploaded table, find the nearest S-PLUS source within 0.0003 degrees.  
Also expand magnitude columns automatically.

```sql
select
    s.ra,
    s.dec,
    s.mag_pstotal*,
    d.ra  as match_ra,
    d.dec as match_dec
from idr6.idr6 as s
join upload.minhatabela as d
  on cone(s.ra, s.dec, d.ra, d.dec, 0.0003);
```

**explanation:**  
- The wildcard `mag_pstotal*` expands to all pstotal magnitude columns.  
- The join condition uses `cone()` to keep only objects close on the sky.  
- Astroql converts this into `q3c_join(...)` internally.

---

## dist() — angular distance

`dist()` returns the great‑circle separation between two sky positions in degrees.

```sql
dist(ra1, dec1, ra2, dec2)
```

### example 3 — distance from a reference point  
**what it does:**  
For each object in `dr3.all_dr3`, compute its distance from (180°, 0°).

```sql
select ra, dec, dist(ra, dec, 180.0, 0.0) as d
from dr3.all_dr3;
```

---

### example 4 — separation after matching  
**what it does:**  
After matching S-PLUS vs your uploaded sources, compute the angular distance between them.

```sql
select
    s.ra, s.dec,
    d.ra  as match_ra,
    d.dec as match_dec,
    dist(s.ra, s.dec, d.ra, d.dec) as sep_deg
from idr6.idr6 as s
join upload.minhatabela as d
  on cone(s.ra, s.dec, d.ra, d.dec, 0.0003);
```

This is typically used to check matching accuracy or filter based on separation.

---

## complete example combining cone() and dist()

**what it does:**  
- First selects objects within a 0.1 deg cone.  
- Then computes the angular distance for each object.

```sql
select
    ra,
    dec,
    dist(ra, dec, 180.0, 0.0) as d
from dr3.all_dr3
where cone(ra, dec, 180.0, 0.0, 0.1);
```

astroql expands patterns → rewrites functions → produces optimized backend sql.