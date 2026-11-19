# Landslide Risk Model: Rain and Terrain

A machine learning project that ranks how likely a landslide is at a place on a given day, using the rain that fell in the previous 14 days and the shape of the terrain around the place.

**Notebook:** `landslide_risk.ipynb` (made in Google Colab)

## Main result

On regions the model had never seen, the model that uses rain and terrain together reached **ROC-AUC 0.77**. A simple rain-only model reached 0.72. Terrain gave a small but steady gain: it was better in all 5 test repeats, by 0.03 to 0.06.

This is a **relative risk ranking**, not a prediction of a real landslide. Please read the limits section.

## Data

| Part | Source |
|---|---|
| Landslide events | NASA Global Landslide Catalog (11,033 events, years up to 2017) |
| Rainfall | Open-Meteo Historical Weather API (daily rain, computer-modelled data) |
| Terrain | Copernicus DEM GLO-90 (90 m height data), read from the free AWS open-data storage |

## Method

1. **Clean the landslide catalogue.** I kept events from 2007 onwards, with a rain-related trigger (rain, downpour, continuous rain, tropical cyclone, monsoon), and with a location accurate to 5 km or better. I removed duplicates (same place and day). This left **4,809 landslides**.
2. **Make "no landslide" examples.** The catalogue has only landslides, so every landslide got two fake examples:
   - *Same place, other day:* the same spot, 31 to 120 days away from the real event.
   - *Other place, same day:* a spot 20 to 150 km away, on the same day.

   Fake examples within 15 km and 30 days of any recorded landslide were thrown away. This gave 14,427 rows, and one row in three is a real landslide.
3. **Rain clues.** From Open-Meteo: rain on the day, over 3, 7 and 14 days, number of wet days, and rain in the earlier part of the window (how wet the ground already was).
4. **Terrain clues.** From a 2 km area around each point: elevation, slope within 500 m, mean slope, steep-slope level (90th percentile), share of ground steeper than 15 degrees, relief (highest minus lowest point), and valley or ridge position.
5. **Remove bad rows.** Rows with missing clues and rows at or below sea level were removed, for real and fake examples alike. Fake points in the sea would let the model cheat. This left **13,233 rows**.
6. **Split by region, not at random.** The map was cut into 5 by 5 degree blocks. Each block went wholly to train, validation or test, and a landslide always stayed with its two fake examples. A random split would put neighbouring slopes on both sides and give scores that are too high. I repeated the split **5 times** with different test regions.
7. **Models.** A 3-day rain total (no model), logistic regression, and LightGBM on rain only, terrain only, and all clues.

Latitude and longitude were not used as clues, so the model cannot memorise places.

## Results (test regions never seen, mean of 5 splits)

| Model | ROC-AUC | Landslide vs same place, other day | Landslide vs other place, same day |
|---|---|---|---|
| 3-day rain total (no model) | 0.723 | 0.838 | 0.570 |
| Logistic regression, all clues | 0.723 | 0.782 | 0.645 |
| LightGBM, rain only | 0.725 | 0.843 | 0.569 |
| LightGBM, terrain only | 0.607 | 0.500 | 0.748 |
| **LightGBM, all clues** | **0.770** | 0.821 | **0.702** |

The spread between the 5 splits was about 0.01 to 0.02 for each score. ROC-AUC 0.77 means that if you pick one landslide row and one non-landslide row at random, the model gives the landslide row the higher score 77% of the time.

The two right-hand columns show what each type of clue does:
- **Rain** separates a landslide from the same spot on a dry day (0.84).
- **Terrain** separates a landslide from a different spot on the same stormy day (0.75). Terrain-only scores exactly 0.50 in the first test, because the two rows share the same ground.
- **Both together** are the best overall.

## What I learned

- Recent rain (especially the last 1 to 3 days) is the strongest clue, with about 40% of the model's importance.
- Terrain clues together give about 40%. Valley or ridge position, slope and steep-ground share all mattered.
- Elevation also scored high. It partly stands for climate and region, so it does not mean that height causes landslides.
- A tree model with only rain clues was no better than a single raw rain number (0.725 against 0.723).
- With only 2,500 landslide groups, the test score was about 0.73. With all 4,809, it was about 0.77. The test regions also changed, so this is likely, but not proven, to be an effect of more data.
- More detailed slope clues alone did not improve the score much. The limit is the data, not the terrain numbers.

## Limits

- **Relative risk only.** Training data has one landslide to two non-landslides, so the scores are not real chances. Real landslides are far rarer.
- **Reporting bias.** The catalogue lists reported landslides, mostly from news. Some countries are covered much better than others, and the number of events rises sharply after 2009 because more sources were added, not because more landslides happened.
- **Rain data is coarse.** It is computer-modelled with a resolution of about 10 to 25 km, so short, intense local storms are smoothed out.
- **Locations are rough.** Catalogue positions can be off by up to 5 km, which blurs the terrain at the true spot.
- **Noisy fake examples.** Some "no landslide" rows may have had a landslide that nobody reported.
- **Missing clues.** Rock type, soil, and land cover are not included, and they matter a lot.
- **Terrain at tile edges.** Places close to the edge of a terrain tile were measured on a partly missing area.
- **Uncertainty is understated.** The 5 repeats reuse the same data, so the real uncertainty is larger than the spread shown.
- The final model was trained on all rows, so it has no test set of its own. The scores above come from the repeated region tests.
- This is a learning project. It must not be used for safety decisions.

## Next steps

- Add rock type and soil maps (this is where geology knowledge can help most).
- Use satellite rainfall with a finer resolution.
- Make a map where a user can pick a place and a date.

## How to run

1. Open `landslide_risk.ipynb` in Google Colab.
2. Run the cells from top to bottom. Results are saved to a Google Drive folder.
3. The rainfall step uses the free Open-Meteo API, which has a daily call limit. Downloading rainfall for all 14,427 rows takes a few days, and the code saves its progress and continues where it stopped.

## Credits and data licences

- Landslide data: NASA Global Landslide Catalog. Kirschbaum, D.B., Adler, R., Hong, Y., Hill, S., and Lerner-Lam, A. (2010). A global landslide catalog for hazard applications: method, results, and limitations. *Natural Hazards*, 52, 561-575.
- Weather data by [Open-Meteo.com](https://open-meteo.com/), used under the CC BY 4.0 license.
- Terrain: Copernicus DEM GLO-90, © DLR e.V. 2010-2014 and © Airbus Defence and Space GmbH 2014-2018, provided under COPERNICUS by the European Union and ESA. Accessed from the AWS Open Data Registry.
