
Link for the nasa's app:
https://worldview.earthdata.nasa.gov/?v=46.701302458835926,27.390732757581176,51.02996525643019,29.80862599389429&l=Reference_Labels_15m(hidden),Reference_Features_15m(hidden),Coastlines_15m,VIIRS_NOAA21_Chlorophyll_a,MODIS_Terra_L2_Chlorophyll_A,MODIS_Aqua_L2_Chlorophyll_A,OCI_PACE_Chlorophyll_a,OCI_PACE_True_Color(hidden),VIIRS_NOAA21_CorrectedReflectance_TrueColor(hidden),VIIRS_NOAA20_CorrectedReflectance_TrueColor(hidden),VIIRS_SNPP_CorrectedReflectance_TrueColor(hidden),MODIS_Aqua_CorrectedReflectance_TrueColor(hidden),MODIS_Terra_CorrectedReflectance_TrueColor&lg=true&t=2026-08-07-T05%3A47%3A22Z
Git repo for the nbasa's thing:
https://github.com/nasa-gibs/worldview/tree/main

The main point is to add the (Add layer thingy) to the KJO app
Need to search:
Database and whether it can be aligned with the Al-Khafji.
Noticed coordinate points-> not sure if possible.


Feasible. Small job. One thing left to check.

The three layers exist and work in your map's projection. MODIS_Terra_L2_Chlorophyll_A, MODIS_Aqua_L2_Chlorophyll_A, OCI_PACE_Chlorophyll_a — all EPSG:3857, PNG, verified live against NASA on 18 Aug.
No database, no account, no cost. Public NASA GIBS tile service, anonymous HTTPS GET. Nothing to host or sync.
Your map is untouched. Same basemap, same vessels, same AOI. The chlorophyll PNGs are transparent except where there's data.
WGS 84 confirmed — the datum risk is closed. Tiles register to the pixel.
Clipping to the polygon is necessary, not cosmetic. One tile spans ~275 km, so without it you'd paint the whole northern Gulf. ~40 lines of canvas code.
Resolution is coarse and always will be. 1 km pixels, capped at zoom 7 — roughly 45 pixels across your AOI. Regional context, not field detail.
Not live. Daily snapshots at satellite overpass, 1–3 hours latency, empty when cloudy. Your vessels are 60 seconds old — label the difference.
You get colours, not numbers. No mg/m³ values for charts or alarms without a different service.

only concern: can a KJO desktop reach gibs.earthdata.nasa.gov?
