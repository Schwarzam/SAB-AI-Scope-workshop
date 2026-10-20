# Query Examples
route: query_examples

# Query Examples
route: examples

# AstroQL Query Guide

**AstroQL** is an astronomy-oriented query language built on top of SQL. It keeps familiar SQL syntax while adding shortcuts for spatial searches, catalog cross-matching, angular distances, and column pattern expansion.

AstroQL queries are translated internally to PostgreSQL and can use spatial backends such as Q3C or pgSphere when suitable indexes are available.

---

## 1. Basic Query

Standard SQL syntax can be used directly.

For example, select bright Gaia DR3 sources:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_g_mean_mag
FROM public.gaia_dr3
WHERE phot_g_mean_mag < 18
LIMIT 100;
```

Standard operations such as `WHERE`, `JOIN`, `LEFT JOIN`, `ORDER BY`, `GROUP BY`, `COUNT`, and `LIMIT` work normally.

---

## 2. Column Wildcards

AstroQL extends SQL wildcards to column names.

### Columns starting with a pattern

```sql
SELECT phot_bp*
FROM public.gaia_dr3;
```

This selects columns beginning with `phot_bp`, such as:

```text
phot_bp_mean_flux
phot_bp_mean_flux_error
phot_bp_mean_flux_over_error
phot_bp_mean_mag
phot_bp_n_obs
...
```

### Columns ending with a pattern

```sql
SELECT *_error
FROM public.gaia_dr3;
```

This selects every column ending in `_error`.

### Columns containing a pattern

```sql
SELECT *parallax*
FROM public.gaia_dr3;
```

This selects every column containing `parallax`.

### Wildcards with aliases

Patterns can be restricted to a table alias:

```sql
SELECT g.*parallax*
FROM public.gaia_dr3 AS g;
```

This is particularly useful for catalogs with many related measurements.

### Alias templates

Wildcard expansions can also generate output aliases:

```sql
SELECT g.phot_bp* AS gaia_*
FROM public.gaia_dr3 AS g;
```

If `phot_bp_mean_mag` is expanded, the resulting output column becomes:

```text
gaia_phot_bp_mean_mag
```

---

# Spatial Queries

AstroQL spatial functions operate on sky coordinates in degrees.

---

## 3. Cone Search

`cone()` selects objects inside a circular region.

Syntax:

```sql
cone(ra, dec, center_ra, center_dec, radius)
```

Example: Gaia DR3 sources within `0.1°` of `(153°, -20°)`:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_g_mean_mag
FROM public.gaia_dr3
WHERE cone(
    ra, dec,
    153.0, -20.0,
    0.1
);
```

A radius in arcseconds can be converted directly.

For example, 4 arcseconds:

```sql
WHERE cone(
    ra, dec,
    153.0, -20.0,
    4.0 / 3600.0
);
```

When a compatible spatial index is available, AstroQL rewrites this to an optimized backend query.

---

## 4. Angular Distance

`dist()` calculates the angular separation between two sky positions.

Syntax:

```sql
dist(ra1, dec1, ra2, dec2)
```

The result is returned in degrees.

Example:

```sql
SELECT
    source_id,
    ra,
    dec,
    dist(ra, dec, 153.0, -20.0) AS separation_deg
FROM public.gaia_dr3;
```

It is commonly combined with a cone search:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_g_mean_mag,
    dist(ra, dec, 153.0, -20.0) AS separation_deg
FROM public.gaia_dr3 AS g
WHERE cone(
    g.ra, g.dec,
    153.0, -20.0,
    0.1
)
ORDER BY separation_deg;
```

---

## 5. Polygon Search

`polygon()` selects objects inside a sky polygon.

Syntax:

```sql
polygon(
    ra, dec,
    ra1, dec1,
    ra2, dec2,
    ra3, dec3,
    ...
)
```

At least three vertices are required.

Example using Gaia DR3:

```sql
SELECT
    source_id,
    ra,
    dec
FROM public.gaia_dr3 AS g
WHERE polygon(
    g.ra, g.dec,
    152.90, -20.10,
    153.10, -20.10,
    153.10, -19.90,
    152.90, -19.90
);
```

Vertices should be provided in order around the polygon boundary.

---

## 6. Box Search

A rectangular sky region can be represented as a four-vertex polygon.

For example:

```text
RA:  152.90 → 153.10
Dec: -20.10 → -19.90
```

can be queried as:

```sql
SELECT
    source_id,
    ra,
    dec
FROM public.gaia_dr3 AS g
WHERE polygon(
    g.ra, g.dec,
    152.90, -20.10,
    153.10, -20.10,
    153.10, -19.90,
    152.90, -19.90
);
```

Using `polygon()` is preferable to treating celestial coordinates as simple Cartesian coordinates for more general sky regions.

---

# Catalog Cross-Matching

## 7. Positional Cross-Match

Catalogs can be matched by their celestial coordinates.

For example, match Gaia DR3 with DELVE DR2 within 1 arcsecond:

```sql
SELECT
    g.source_id,
    g.ra,
    g.dec,
    d.id AS delve_id,
    d.ra AS delve_ra,
    d.dec AS delve_dec,
    dist(g.ra, g.dec, d.ra, d.dec) AS separation_deg
FROM public.gaia_dr3 AS g
JOIN public.delve_dr2_cleaned AS d
    ON cone(
        g.ra, g.dec,
        d.ra, d.dec,
        1.0 / 3600.0
    );
```

AstroQL recognizes the two-table spatial comparison and can translate it into an optimized spatial join.

---

## 8. Cross-Match Gaia with APOGEE

Both catalogs contain sky positions, so they can be matched directly:

```sql
SELECT
    g.source_id,
    g.ra,
    g.dec,
    a.apogee_id,
    a.ra AS apogee_ra,
    a.dec AS apogee_dec,
    a.teff,
    a.logg,
    a.fe_h,
    dist(g.ra, g.dec, a.ra, a.dec) AS separation_deg
FROM public.gaia_dr3 AS g
JOIN public.apogee_synspec_dr17 AS a
    ON cone(
        g.ra, g.dec,
        a.ra, a.dec,
        1.0 / 3600.0
    );
```

---

## 9. Cross-Match by ID

When catalogs share an identifier, use a normal SQL join.

For example, Gaia DR3 and the Gaia geometric-distance catalog share `source_id`:

```sql
SELECT
    g.source_id,
    g.ra,
    g.dec,
    g.parallax,
    d.r_med_geo,
    d.r_lo_geo,
    d.r_hi_geo
FROM public.gaia_dr3 AS g
JOIN public.geodist_gaia_dr3 AS d
    ON g.source_id = d.source_id
LIMIT 100;
```

Use ID matching when a reliable common identifier exists.

---

# Scientific Query Examples

## 10. Gaia Photometry

Select Gaia photometric measurements:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_g_mean_mag,
    phot_bp_mean_mag,
    phot_rp_mean_mag,
    bp_rp
FROM public.gaia_dr3
LIMIT 1000;
```

Using AstroQL wildcards:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_*_mean_mag
FROM public.gaia_dr3
LIMIT 1000;
```

If the wildcard expression matches the available column names, AstroQL expands them automatically.

---

## 11. Gaia Astrometry

Select parallax and proper-motion information:

```sql
SELECT
    source_id,
    ra,
    dec,
    parallax,
    parallax_error,
    pmra,
    pmra_error,
    pmdec,
    pmdec_error,
    ruwe
FROM public.gaia_dr3
WHERE parallax IS NOT NULL;
```

A shorter AstroQL version can use:

```sql
SELECT
    source_id,
    ra,
    dec,
    *parallax*,
    pmra,
    pmra_error,
    pmdec,
    pmdec_error,
    ruwe
FROM public.gaia_dr3;
```

---

## 12. Gaia Stellar Parameters

Gaia DR3 provides stellar parameters derived by GSP-Phot:

```sql
SELECT
    source_id,
    ra,
    dec,
    teff_gspphot,
    logg_gspphot,
    mh_gspphot,
    distance_gspphot
FROM public.gaia_dr3
WHERE teff_gspphot IS NOT NULL
LIMIT 1000;
```

You can select the associated uncertainty ranges using wildcards:

```sql
SELECT
    source_id,
    teff_gspphot*,
    logg_gspphot*,
    mh_gspphot*
FROM public.gaia_dr3
LIMIT 1000;
```

---

## 13. Gaia Galaxy and Quasar Probabilities

Gaia DR3 contains classification probabilities:

```sql
SELECT
    source_id,
    ra,
    dec,
    classprob_dsc_combmod_galaxy,
    classprob_dsc_combmod_quasar,
    classprob_dsc_combmod_star
FROM public.gaia_dr3
WHERE classprob_dsc_combmod_galaxy > 0.98;
```

For galaxies around a specific position:

```sql
SELECT
    source_id,
    ra,
    dec,
    classprob_dsc_combmod_galaxy
FROM public.gaia_dr3
WHERE cone(
    ra, dec,
    153.0, -20.0,
    0.5
)
AND classprob_dsc_combmod_galaxy > 0.98;
```

---

## 14. DELVE Multi-Band Photometry

DELVE DR2 provides photometry in multiple bands.

```sql
SELECT
    id,
    ra,
    dec,
    mag_psf_g,
    mag_psf_r,
    mag_psf_i,
    mag_psf_z
FROM public.delve_dr2_cleaned
LIMIT 1000;
```

With AstroQL wildcards:

```sql
SELECT
    id,
    ra,
    dec,
    mag_psf*
FROM public.delve_dr2_cleaned
LIMIT 1000;
```

This expands all columns beginning with `mag_psf`.

To include magnitude uncertainties:

```sql
SELECT
    id,
    ra,
    dec,
    mag_psf*,
    magerr_psf*
FROM public.delve_dr2_cleaned
LIMIT 1000;
```

---

## 15. DELVE Morphology

The DELVE catalog contains several morphology-related measurements:

```sql
SELECT
    id,
    ra,
    dec,
    class_star_g,
    class_star_r,
    class_star_i,
    class_star_z,
    spread_model_g,
    spread_model_r,
    spread_model_i,
    spread_model_z
FROM public.delve_dr2_cleaned;
```

Wildcards make this shorter:

```sql
SELECT
    id,
    ra,
    dec,
    class_star*,
    spread_model*
FROM public.delve_dr2_cleaned;
```

---

## 16. DELVE Photometric Redshifts

The `public.delve_dr2_photoz` table contains photometric-redshift measurements.

```sql
SELECT
    id,
    ra,
    dec,
    mag_auto_g,
    mag_auto_r,
    z_phot_median,
    z_phot_l68,
    z_phot_u68,
    z_phot_odds
FROM public.delve_dr2_photoz
WHERE z_phot_odds > 0.8
LIMIT 1000;
```

AstroQL wildcards can select all redshift-related columns:

```sql
SELECT
    id,
    ra,
    dec,
    z_phot*
FROM public.delve_dr2_photoz
LIMIT 1000;
```

---

## 17. APOGEE Stellar Abundances

APOGEE DR17 provides stellar parameters and elemental abundances.

```sql
SELECT
    apogee_id,
    ra,
    dec,
    teff,
    logg,
    fe_h,
    mg_fe,
    si_fe,
    ca_fe
FROM public.apogee_synspec_dr17
WHERE snr > 100
LIMIT 1000;
```

AstroQL can select abundance families using column patterns:

```sql
SELECT
    apogee_id,
    ra,
    dec,
    *_fe
FROM public.apogee_synspec_dr17
LIMIT 1000;
```

This selects columns whose names end in `_fe`.

---

## 18. Milliquas Quasars

Select known quasars from the Milliquas catalog:

```sql
SELECT
    name,
    ra,
    dec,
    z,
    type,
    bmag,
    rmag
FROM public.milliquas
WHERE z IS NOT NULL
LIMIT 1000;
```

Quasars around a sky position:

```sql
SELECT
    name,
    ra,
    dec,
    z,
    type,
    dist(ra, dec, 180.0, 0.0) AS separation_deg
FROM public.milliquas
WHERE cone(
    ra, dec,
    180.0, 0.0,
    1.0
)
ORDER BY separation_deg;
```

---

## 19. ZTF Light-Curve Data

`public.ztf_dr22` stores light-curve measurements in arrays.

```sql
SELECT
    id,
    ra,
    dec,
    filterid,
    nepochs,
    hmjd,
    mag,
    magerr
FROM public.ztf_dr22
WHERE nepochs > 50
LIMIT 100;
```

A spatial search can be applied normally:

```sql
SELECT
    id,
    ra,
    dec,
    filterid,
    nepochs,
    hmjd,
    mag,
    magerr
FROM public.ztf_dr22
WHERE cone(
    ra, dec,
    153.0, -20.0,
    0.1
)
AND nepochs > 20;
```

---

# Filtering

## 20. Magnitude Cuts

Gaia example:

```sql
WHERE phot_g_mean_mag < 18;
```

DELVE example:

```sql
WHERE mag_psf_r < 21;
```

Multiple conditions:

```sql
WHERE mag_psf_g < 22
  AND mag_psf_r < 21;
```

---

## 21. Proper-Motion Cuts

```sql
SELECT
    source_id,
    ra,
    dec,
    pmra,
    pmdec
FROM public.gaia_dr3
WHERE pmra BETWEEN -5.0 AND 5.0
  AND pmdec BETWEEN -5.0 AND 5.0;
```

---

## 22. Parallax Quality Cuts

```sql
SELECT
    source_id,
    ra,
    dec,
    parallax,
    parallax_error,
    parallax_over_error
FROM public.gaia_dr3
WHERE parallax_over_error > 5;
```

---

## 23. Color Cuts

Gaia colors can be selected directly:

```sql
SELECT
    source_id,
    ra,
    dec,
    bp_rp
FROM public.gaia_dr3
WHERE bp_rp BETWEEN 0.5 AND 2.0;
```

Colors can also be calculated:

```sql
SELECT
    source_id,
    phot_bp_mean_mag - phot_rp_mean_mag AS bp_minus_rp
FROM public.gaia_dr3
WHERE
    (phot_bp_mean_mag - phot_rp_mean_mag)
    BETWEEN 0.5 AND 2.0;
```

---

## 24. Combining Spatial and Physical Filters

AstroQL spatial primitives can be freely combined with ordinary SQL conditions.

Example: nearby Gaia stars with good astrometry:

```sql
SELECT
    source_id,
    ra,
    dec,
    phot_g_mean_mag,
    parallax,
    pmra,
    pmdec,
    ruwe,
    dist(ra, dec, 153.0, -20.0) AS separation_deg
FROM public.gaia_dr3 AS g
WHERE cone(
    g.ra, g.dec,
    153.0, -20.0,
    0.5
)
AND phot_g_mean_mag < 18
AND ruwe < 1.4
AND parallax_over_error > 5
ORDER BY separation_deg;
```

---

# Large Queries

## 25. Counting Objects

Use `COUNT()` when only the number of matching objects is required:

```sql
SELECT COUNT(source_id)
FROM public.gaia_dr3
WHERE phot_g_mean_mag < 18;
```

Count sources inside a spatial region:

```sql
SELECT COUNT(source_id)
FROM public.gaia_dr3
WHERE cone(
    ra, dec,
    153.0, -20.0,
    1.0
);
```

---

## 26. Limiting Results

During query development, use `LIMIT`:

```sql
SELECT *
FROM public.gaia_dr3
LIMIT 1000;
```

This avoids unnecessarily transferring large result sets.

---

## 27. Sorting by Distance

A common spatial-query pattern is:

```sql
SELECT
    source_id,
    ra,
    dec,
    dist(ra, dec, 153.0, -20.0) AS separation_deg
FROM public.gaia_dr3
WHERE cone(
    ra, dec,
    153.0, -20.0,
    0.1
)
ORDER BY separation_deg;
```

This returns sources from nearest to farthest from the reference position.

---

## 28. Combining Catalogs by ID and Position

AstroQL and regular SQL can be combined in the same query.

For example, first join Gaia to its geometric-distance catalog using `source_id`, then spatially restrict the result:

```sql
SELECT
    g.source_id,
    g.ra,
    g.dec,
    g.parallax,
    d.r_med_geo,
    d.r_lo_geo,
    d.r_hi_geo
FROM public.gaia_dr3 AS g
JOIN public.geodist_gaia_dr3 AS d
    ON g.source_id = d.source_id
WHERE cone(
    g.ra, g.dec,
    153.0, -20.0,
    0.5
);
```

---

# AstroQL Quick Reference

| AstroQL | Purpose |
|---|---|
| `cone(ra, dec, ra0, dec0, radius)` | Circular sky search |
| `polygon(ra, dec, ra1, dec1, ...)` | Polygon or rectangular sky region |
| `dist(ra1, dec1, ra2, dec2)` | Angular separation in degrees |
| `prefix*` | Columns starting with `prefix` |
| `*suffix` | Columns ending with `suffix` |
| `*text*` | Columns containing `text` |
| `alias.*text*` | Wildcard restricted to one table |
| `pattern* AS name_*` | Wildcard expansion with generated aliases |
| `LIMIT n` | Limit returned rows |
| `COUNT(...)` | Count matching objects |

---

## Units

AstroQL sky coordinates and angular distances are expressed in **degrees**.

```text
1 arcmin = 1 / 60 degree
1 arcsec = 1 / 3600 degree
```

For example:

```sql
-- 5 arcminutes
5.0 / 60.0

-- 2 arcseconds
2.0 / 3600.0
```

A 2-arcsecond cone search is:

```sql
WHERE cone(
    ra, dec,
    153.0, -20.0,
    2.0 / 3600.0
);
```