# חלופות קוד פתוח ל‑Google Maps API

מה שצריך בשביל הכלי הוא שני דברים נפרדים, ולכל אחד יש פרויקט קוד פתוח בוגר:

1. **גיאוקודינג** — להפוך כתובת ("העליה 38 נהריה") לנקודה על המפה.
2. **ניתוב / מטריצת זמנים** — כמה דקות נהיגה יש בין נקודה א' לנקודה ב'.

כל אלה בנויים על **OpenStreetMap** — מסד נתוני מפות פתוח (רישיון ODbL), שבישראל מכוסה טוב מאוד.

---

## מה מותקן עכשיו בכלי (ברירת מחדל)

| רכיב | פרויקט | כתובת | מפתח | עלות |
|---|---|---|---|---|
| גיאוקודינג | **Photon** (komoot) | photon.komoot.io | לא צריך | חינם |
| מטריצת זמנים | **OSRM** — Open Source Routing Machine | router.project-osrm.org | לא צריך | חינם |

שתי הבקשות עוברות מהדפדפן שלך ישירות לשרתים האלה. שיבוץ של 12 בתי ספר ו‑5 מנחות = 17 בקשות גיאוקודינג + **בקשה אחת** למטריצה שמחזירה את כל 60 הזמנים בבת אחת. התוצאות נשמרות בקובץ ה‑JSON, כך שכל כתובת מחושבת פעם אחת בלבד.

---

## הטבלה המלאה — מה קיים בשוק

### מנועי ניתוב (Routing / Matrix)

| פרויקט | רישיון | מטריצה | שרת ציבורי חינם | מתי לבחור |
|---|---|---|---|---|
| **OSRM** | BSD‑2 | `/table` — מהיר מאוד | router.project-osrm.org | ברירת המחדל. הכי מהיר למטריצות גדולות |
| **Valhalla** | MIT | `sources_to_targets` | valhalla1.openstreetmap.de (FOSSGIS) | אם צריך חלונות זמן, סוגי רכב, isochrones |
| **GraphHopper** | Apache‑2 | Matrix API | graphhopper.com (מוגבל) | ג'אווה, קל להרים, אופטימיזציית מסלולים |
| **OpenRouteService** | GPL | Matrix | openrouteservice.org — **מפתח חינם**, ~500 בקשות ביום | אם רוצים מפתח וזהות אבל בלי לשלם |

### גיאוקודרים

| פרויקט | רישיון | שרת ציבורי | הערות |
|---|---|---|---|
| **Photon** | Apache‑2 | photon.komoot.io | מבוסס Elasticsearch, סובלני לשגיאות כתיב, CORS פתוח |
| **Nominatim** | GPL | nominatim.openstreetmap.org | הגיאוקודר הרשמי של OSM. **מגבלה: בקשה אחת בשנייה**, חובה לציין Referer/User‑Agent. [מדיניות השימוש](https://operations.osmfoundation.org/policies/nominatim/) |
| **Pelias** | MIT | — | לרוב מריצים עצמאית, איכות גבוהה |

---

## למה לא "סקרייפינג של Google Maps"

יש פרויקטים כאלה ב‑GitHub, אבל בשביל המקרה הזה זו הדרך הגרועה:

- **מנוגד לתנאי השימוש** של Google — סעיף 3.2.3(a) אוסר במפורש על גירוד תוצאות. שימוש עסקי בארגון חשוף לתביעה.
- **שביר** — Google משנה את מבנה ה‑HTML כל כמה שבועות, וכל שינוי שובר את הסקרייפר.
- **נחסם** — בקשות אוטומטיות מזוהות תוך עשרות בקשות; צריך פרוקסים מסתובבים ופתרון CAPTCHA.
- **מיותר** — OSM ו‑OSRM נותנים בדיוק את אותו מספר, חוקית, בבקשה אחת. בבדיקה שעשיתי על הכתובות שלך הפער מול Google היה דקות בודדות.

---

## אם רוצים עצמאות מלאה — שרת משלכם

מפת ישראל שלמה שוקלת כ‑120MB ורצה בנוחות על כל מחשב או VPS זול. בכלי יש לשונית "שרת עצמי" שבה מזינים את הכתובת המקומית.

```bash
# 1. מורידים את מפת ישראל
wget https://download.geofabrik.de/asia/israel-and-palestine-latest.osm.pbf

# 2. מעבדים אותה (פעם אחת, כמה דקות)
docker run -t -v "${PWD}:/data" osrm/osrm-backend \
  osrm-extract -p /opt/car.lua /data/israel-and-palestine-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend \
  osrm-partition /data/israel-and-palestine-latest.osrm
docker run -t -v "${PWD}:/data" osrm/osrm-backend \
  osrm-customize /data/israel-and-palestine-latest.osrm

# 3. מריצים את השרת
docker run -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend \
  osrm-routed --algorithm mld /data/israel-and-palestine-latest.osrm
```

בדיקה שהכל עובד:
```
http://localhost:5000/table/v1/driving/35.26,32.71;35.09,32.50?annotations=duration
```

לגיאוקודר מקומי (אופציונלי):
```bash
docker run -it -e PBF_URL=https://download.geofabrik.de/asia/israel-and-palestine-latest.osm.pbf \
  -p 8080:8080 mediagis/nominatim:4.4
```

יתרונות: בלי מגבלות קצב, בלי תלות בשרת חיצוני, המידע לא יוצא מהארגון, ועובד גם ברשת סגורה.

---

## קרדיטים ורישוי

נתוני המפות: **© OpenStreetMap contributors**, רישיון [ODbL](https://opendatacommons.org/licenses/odbl/). חובה להציג את הקרדיט — הוא כבר מופיע בכלי.

**קישורים:**

- OSRM — https://github.com/Project-OSRM/osrm-backend
- Valhalla — https://github.com/valhalla/valhalla
- GraphHopper — https://github.com/graphhopper/graphhopper
- OpenRouteService — https://github.com/GIScience/openrouteservice
- Photon — https://github.com/komoot/photon
- Nominatim — https://github.com/osm-search/Nominatim
- מפות להורדה — https://download.geofabrik.de/asia/israel-and-palestine.html
