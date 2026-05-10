(function () {

    // ===================================================
    //  Cinemana SkyStream Plugin
    //  تحويل من CloudStream Kotlin إلى SkyStream JavaScript
    // ===================================================

    const BASE_URL = manifest.baseUrl || "https://cinemana.shabakaty.cc";
    const API = `${BASE_URL}/api/android`;

    // ===================================================
    //  دوال مساعدة
    // ===================================================

    async function fetchJSON(url) {
        try {
            const res = await fetch(url, {
                headers: { "Referer": BASE_URL }
            });
            if (!res.ok) return null;
            return await res.json();
        } catch (e) {
            return null;
        }
    }

    // تحويل عنصر JSON من API إلى MultimediaItem
    function toMultimediaItem(item) {
        if (!item || !item.nb) return null;

        const isMovie = String(item.kind) !== "2";
        const type = isMovie ? "movie" : "series";

        const categories = (item.categories || [])
            .map(c => c.en_title || c.ar_title)
            .filter(Boolean);

        const cast = (item.actorsInfo || [])
            .filter(a => a.name)
            .map(a => new Actor({
                name: a.name,
                image: a.staff_img_thumb || a.staff_img || ""
            }));

        return new MultimediaItem({
            title: item.en_title || item.ar_title || "بدون عنوان",
            url: `${BASE_URL}/video/${item.nb}`,
            posterUrl: item.imgObjUrl || item.img || "",
            type: type,
            year: item.year ? parseInt(item.year) : undefined,
            score: item.stars ? parseFloat(item.stars) : undefined,
            description: item.ar_content || item.en_content || "",
            cast: cast.length ? cast : undefined,
        });
    }

    // ===================================================
    //  1. getHome — الصفحة الرئيسية
    // ===================================================

    async function getHome(cb) {
        try {
            // أحدث الإضافات → Hero Carousel
            const newlyData = await fetchJSON(`${API}/newlyVideosItems/level/0/offset/12/page/1/`);
            const trending = (newlyData || [])
                .map(toMultimediaItem)
                .filter(Boolean);

            // أفلام الأكثر مشاهدة
            const moviesData = await fetchJSON(
                `${BASE_URL}/api/android/video/V/2?videoKind=1&langNb=&itemsPerPage=24&pageNumber=0&level=0&sortParam=views_desc`
            );
            const topMovies = (moviesData || [])
                .map(toMultimediaItem)
                .filter(Boolean);

            // مسلسلات الأكثر مشاهدة
            const seriesData = await fetchJSON(
                `${BASE_URL}/api/android/video/V/2?videoKind=2&langNb=&itemsPerPage=24&pageNumber=0&level=0&sortParam=views_desc`
            );
            const topSeries = (seriesData || [])
                .map(toMultimediaItem)
                .filter(Boolean);

            // أعلى تقييم أفلام
            const topRatedMovies = await fetchJSON(
                `${BASE_URL}/api/android/video/V/2?videoKind=1&langNb=&itemsPerPage=24&pageNumber=0&level=0&sortParam=stars_desc`
            );
            const ratedMovies = (topRatedMovies || [])
                .map(toMultimediaItem)
                .filter(Boolean);

            // أعلى تقييم مسلسلات
            const topRatedSeries = await fetchJSON(
                `${BASE_URL}/api/android/video/V/2?videoKind=2&langNb=&itemsPerPage=24&pageNumber=0&level=0&sortParam=stars_desc`
            );
            const ratedSeries = (topRatedSeries || [])
                .map(toMultimediaItem)
                .filter(Boolean);

            const data = {};
            if (trending.length)   data["Trending"]                  = trending;
            if (topMovies.length)  data["أفلام - الأكثر مشاهدة"]    = topMovies;
            if (topSeries.length)  data["مسلسلات - الأكثر مشاهدة"]  = topSeries;
            if (ratedMovies.length) data["أفلام - أعلى تقييم"]      = ratedMovies;
            if (ratedSeries.length) data["مسلسلات - أعلى تقييم"]    = ratedSeries;

            cb({ success: true, data });
        } catch (e) {
            cb({ success: false, error: e.message });
        }
    }

    // ===================================================
    //  2. search — البحث
    // ===================================================

    async function search(query, cb) {
        try {
            const encoded = encodeURIComponent(query);
            const currentYear = new Date().getFullYear();
            const yearRange = `1900,${currentYear}`;

            const moviesUrl = `${API}/AdvancedSearch?level=0&videoTitle=${encoded}&staffTitle=${encoded}&year=${yearRange}&page=0&type=movies&itemsPerPage=30`;
            const seriesUrl = `${API}/AdvancedSearch?level=0&videoTitle=${encoded}&staffTitle=${encoded}&year=${yearRange}&page=0&type=series&itemsPerPage=30`;

            const [moviesData, seriesData] = await Promise.all([
                fetchJSON(moviesUrl),
                fetchJSON(seriesUrl)
            ]);

            const movies = (moviesData || []).map(toMultimediaItem).filter(Boolean);
            const series = (seriesData || []).map(toMultimediaItem).filter(Boolean);

            // دمج متناوب (فيلم، مسلسل، فيلم، ...)
            const interleaved = [];
            const maxLen = Math.max(movies.length, series.length);
            for (let i = 0; i < maxLen; i++) {
                if (i < movies.length) interleaved.push(movies[i]);
                if (i < series.length) interleaved.push(series[i]);
            }

            // ترتيب حسب التطابق مع الاستعلام
            function scoreMatch(title, q) {
                if (!title) return 0;
                const t = title.toLowerCase();
                const ql = q.toLowerCase().trim();
                if (t === ql) return 100;
                if (t.startsWith(ql)) return 80;
                if (t.includes(ql)) return 60;
                const tokens = ql.split(/\s+/).filter(Boolean);
                return 40 + tokens.filter(tok => t.includes(tok)).length;
            }

            const sorted = interleaved
                .map((item, idx) => ({ item, score: scoreMatch(item.title, query), idx }))
                .sort((a, b) => b.score - a.score || a.idx - b.idx)
                .map(x => x.item);

            // إزالة المكررات
            const seen = new Set();
            const results = sorted.filter(item => {
                const key = item.url + item.title;
                if (seen.has(key)) return false;
                seen.add(key);
                return true;
            });

            cb({ success: true, data: results });
        } catch (e) {
            cb({ success: false, error: e.message });
        }
    }

    // ===================================================
    //  3. load — تفاصيل الفيلم/المسلسل
    // ===================================================

    async function load(url, cb) {
        try {
            const id = url.split("/").pop();
            const detailsUrl = `${BASE_URL}/api/android/allVideoInfo/id/${id}`;
            const details = await fetchJSON(detailsUrl);

            if (!details) {
                cb({ success: false, error: "فشل تحميل التفاصيل" });
                return;
            }

            const isMovie = String(details.kind) !== "2";

            const categories = (details.categories || [])
                .map(c => c.en_title || c.ar_title)
                .filter(Boolean);

            const cast = (details.actorsInfo || [])
                .filter(a => a.name)
                .map(a => new Actor({
                    name: a.name,
                    image: a.staff_img_thumb || a.staff_img || ""
                }));

            let episodes = [];

            if (!isMovie) {
                // جلب حلقات المسلسل
                const seasonsUrl = `${BASE_URL}/api/android/videoSeason/id/${id}`;
                const episodesData = await fetchJSON(seasonsUrl);

                if (episodesData && episodesData.length) {
                    // تنظيم الحلقات حسب الموسم
                    const seasonsMap = {};
                    for (const ep of episodesData) {
                        if (!ep.nb || !ep.en_title) continue;
                        const seasonNum = parseInt(ep.season) || 1;
                        const epNum = parseInt(ep.episodeNummer) || 1;
                        if (!seasonsMap[seasonNum]) seasonsMap[seasonNum] = [];
                        seasonsMap[seasonNum].push(new Episode({
                            name: `الموسم ${seasonNum} - الحلقة ${epNum}`,
                            url: `${BASE_URL}/video/${ep.nb}`,
                            season: seasonNum,
                            episode: epNum,
                        }));
                    }
                    // فرز وتجميع
                    const sortedSeasons = Object.keys(seasonsMap).map(Number).sort((a, b) => a - b);
                    for (const sNum of sortedSeasons) {
                        const eps = seasonsMap[sNum].sort((a, b) => a.episode - b.episode);
                        episodes.push(...eps);
                    }
                }
            }

            const item = new MultimediaItem({
                title: details.en_title || details.ar_title || "بدون عنوان",
                url: url,
                posterUrl: details.imgObjUrl || details.img || "",
                type: isMovie ? "movie" : "series",
                year: details.year ? parseInt(details.year) : undefined,
                score: details.stars ? parseFloat(details.stars) : undefined,
                description: details.ar_content || details.en_content || "",
                cast: cast.length ? cast : undefined,
                episodes: episodes.length ? episodes : undefined,
            });

            cb({ success: true, data: item });
        } catch (e) {
            cb({ success: false, error: e.message });
        }
    }

    // ===================================================
    //  4. loadStreams — روابط التشغيل
    // ===================================================

    async function loadStreams(url, cb) {
        try {
            const id = url.split("/").pop();
            const videosUrl = `${API}/transcoddedFiles/id/${id}`;
            const videoResponse = await fetchJSON(videosUrl);

            const streams = [];

            if (videoResponse && videoResponse.length) {
                // عكس الترتيب لإظهار الجودة الأعلى أولاً
                const reversed = [...videoResponse].reverse();
                for (const v of reversed) {
                    if (!v.videoUrl) continue;
                    streams.push(new StreamResult({
                        url: v.videoUrl,
                        quality: v.resolution || "Default",
                        headers: { "Referer": BASE_URL }
                    }));
                }
            }

            // جلب الترجمات
            const detailsUrl = `${API}/allVideoInfo/id/${id}`;
            const details = await fetchJSON(detailsUrl);
            const subtitles = [];

            if (details && details.translations) {
                const sortOrder = { ass: 0, vtt: 1, srt: 2 };
                const sorted = [...details.translations].sort((a, b) => {
                    const ea = sortOrder[(a.extention || "").toLowerCase()] ?? 3;
                    const eb = sortOrder[(b.extention || "").toLowerCase()] ?? 3;
                    return ea - eb;
                });
                for (const sub of sorted) {
                    if (!sub.file) continue;
                    const ext = (sub.extention || "").toLowerCase();
                    const lang = ext === "ass" ? "Arabic" : (sub.name || "Arabic");
                    subtitles.push({ url: sub.file, label: lang, lang: lang });
                }
            }

            // إضافة الترجمات لأول stream
            if (streams.length && subtitles.length) {
                streams[0].subtitles = subtitles;
            }

            cb({ success: true, data: streams });
        } catch (e) {
            cb({ success: false, error: e.message });
        }
    }

    // ===================================================
    //  تصدير الدوال إلى SkyStream
    // ===================================================

    globalThis.getHome = getHome;
    globalThis.search = search;
    globalThis.load = load;
    globalThis.loadStreams = loadStreams;

})();
