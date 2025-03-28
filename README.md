# Recreate GA4 native events

```javascript
(function () {
    // Initialize dataLayer
    window.dataLayer = window.dataLayer || [];

    function sendEvent(eventName, params = {}) {
        window.dataLayer.push({ event: eventName, ...params });
    }

    // Detect first visit
    function trackFirstVisit() {
        if (!localStorage.getItem("first_visit")) {
            sendEvent("first_visit");
            localStorage.setItem("first_visit", "true");
        }
    }

    // Start session (if inactive for 30 min)
    function trackSession() {
        let lastSessionTime = sessionStorage.getItem("session_start");
        let now = Date.now();
        if (!lastSessionTime || now - lastSessionTime > 30 * 60 * 1000) {
            sendEvent("session_start");
            sessionStorage.setItem("session_start", now);
        }
    }

    // User engagement (every 10 sec)
    function trackUserEngagement() {
        setInterval(() => sendEvent("user_engagement"), 10000);
    }

    // Page view
    function trackPageView() {
        sendEvent("page_view", { url: window.location.href });
    }

    // Outbound clicks
    function trackOutboundClicks() {
        document.addEventListener("click", function (event) {
            let link = event.target.closest("a");
            if (link && link.href && !link.href.includes(location.hostname)) {
                sendEvent("click", { outbound_url: link.href });
            }
        });
    }

    // Internal search
    function trackSearch() {
        let params = new URLSearchParams(window.location.search);
        let searchTerm = params.get("q") || params.get("s") || params.get("search");
        if (searchTerm) {
            sendEvent("view_search_results", { search_term: searchTerm });
        }
    }

    // Scroll at 90%
    function trackScroll() {
        let scrolled = false;
        window.addEventListener("scroll", function () {
            if (!scrolled && window.innerHeight + window.scrollY >= document.documentElement.scrollHeight * 0.9) {
                sendEvent("scroll");
                scrolled = true;
            }
        });
    }

    // File downloads
    function trackFileDownload() {
        document.addEventListener("click", function (event) {
            let link = event.target.closest("a");
            if (link && link.href && link.download) {
                sendEvent("file_download", { file_name: link.href.split("/").pop() });
            }
        });
    }

    // Form start and submission detection
    function trackForms() {
        document.addEventListener("focusin", function (event) {
            if (event.target.closest("form")) {
                sendEvent("form_start");
            }
        });

        document.addEventListener("submit", function (event) {
            sendEvent("form_submit");
        });
    }

    // HTML5 video tracking
    function trackHTML5Videos() {
        document.querySelectorAll("video").forEach((video) => {
            video.addEventListener("play", () => sendEvent("video_start", { video_url: video.currentSrc }));
            video.addEventListener("ended", () => sendEvent("video_complete", { video_url: video.currentSrc }));
            video.addEventListener("timeupdate", () => {
                let percent = (video.currentTime / video.duration) * 100;
                if ([10, 25, 50, 75].includes(Math.round(percent))) {
                    sendEvent("video_progress", { video_url: video.currentSrc, progress: Math.round(percent) });
                }
            });
        });
    }

    // YouTube video tracking
    function trackYouTubeVideos() {
        if (!window.YT || !YT.Player) return;

        document.querySelectorAll("iframe[src*='youtube.com']").forEach((iframe) => {
            let player = new YT.Player(iframe, {
                events: {
                    onStateChange: function (event) {
                        if (event.data == YT.PlayerState.PLAYING) sendEvent("video_start", { video_url: iframe.src });
                        if (event.data == YT.PlayerState.ENDED) sendEvent("video_complete", { video_url: iframe.src });
                    },
                },
            });
        });
    }

    // Load YouTube API if necessary
    function loadYouTubeAPI() {
        if (document.querySelector("iframe[src*='youtube.com']")) {
            let tag = document.createElement("script");
            tag.src = "https://www.youtube.com/iframe_api";
            document.body.appendChild(tag);
            window.onYouTubeIframeAPIReady = trackYouTubeVideos;
        }
    }

    // ================== FUNCTION CALLS ================== //
    trackFirstVisit();      // first_visit
    trackSession();         // session_start
    trackUserEngagement();  // user_engagement
    trackPageView();        // page_view
    trackOutboundClicks();  // click
    trackSearch();          // view_search_results
    trackScroll();          // scroll
    trackFileDownload();    // file_download
    trackForms();           // form_start, form_submit
    trackHTML5Videos();     // video_start, video_progress, video_complete
    loadYouTubeAPI();       // video_start, video_complete (YouTube)
})();
```