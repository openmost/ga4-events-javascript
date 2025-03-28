# Recreate GA4 native events

```javascript
(function () {
    // Initialize dataLayer
    window.dataLayer = window.dataLayer || [];

    function sendEvent(eventName, params = {}) {
        const defaultParams = {
            language: navigator.language || navigator.userLanguage,
            page_location: window.location.href,
            page_referrer: document.referrer,
            page_title: document.title,
            screen_resolution: `${window.screen.width}x${window.screen.height}`
            user_agent: navigator.userAgent,
            operating_system: navigator.platform,
            timestamp: new Date().toISOString(),  
        };

        window.dataLayer.push({ event: eventName, ...defaultParams, ...params });
    }

    // Track first visit (check if it's the first visit)
    function trackFirstVisit() {
        if (!localStorage.getItem("first_visit")) {
            sendEvent("first_visit", {
                engagement_time_msec: 0
            });
            localStorage.setItem("first_visit", "true");
        }
    }

    // Track session (if inactive for 30 min)
    function trackSession() {
        let lastSessionTime = sessionStorage.getItem("session_start");
        let now = Date.now();
        if (!lastSessionTime || now - lastSessionTime > 30 * 60 * 1000) {
            sendEvent("session_start", {
                engagement_time_msec: now - lastSessionTime,
                session_id: now // Using timestamp as a session ID
            });
            sessionStorage.setItem("session_start", now);
        }
    }

    // Track user engagement (every 10 seconds)
    function trackUserEngagement() {
        setInterval(() => sendEvent("user_engagement", {
            engagement_time_msec: 10000,
            page_location: window.location.href
        }), 10000);
    }

    // Track page view
    function trackPageView() {
        sendEvent("page_view");
    }

    // Track outbound link clicks
    function trackOutboundClicks() {
        document.addEventListener("click", function (event) {
            let link = event.target.closest("a");
            if (link && link.href) {
                const isOutbound = new URL(link.href).hostname !== window.location.hostname;  // Dynamically check if the link is outbound
                if (isOutbound) {
                    sendEvent("click", {
                        link_classes: link.className,
                        link_domain: new URL(link.href).hostname,
                        link_id: link.id || null,
                        link_url: link.href,
                        outbound: true
                    });
                }
            }
        });
    }

    // Track internal search terms
    function trackSearch() {
        let params = new URLSearchParams(window.location.search);
        let searchTerm = params.get("q") || params.get("s") || params.get("search") || params.get("query") || params.get("keyword");
        if (searchTerm) {
            // Check if the search term is unique for the session using localStorage
            let searchHistory = JSON.parse(localStorage.getItem("searchHistory")) || [];
            let isUniqueSearchTerm = !searchHistory.includes(searchTerm);

            if (isUniqueSearchTerm) {
                searchHistory.push(searchTerm);
                localStorage.setItem("searchHistory", JSON.stringify(searchHistory));
            }

            sendEvent("view_search_results", {
                search_term: searchTerm,
                unique_search_term: isUniqueSearchTerm ? 1 : 0, // Only set to 1 if it's unique in this session
                search_page: window.location.href
            });
        }
    }

    // Track scroll events at 90% of the page height
    function trackScroll() {
        let scrolled = false;
        window.addEventListener("scroll", function () {
            if (!scrolled && window.innerHeight + window.scrollY >= document.documentElement.scrollHeight * 0.9) {
                sendEvent("scroll", {
                    scroll_percentage: 90,
                    engagement_time_msec: Date.now() - window.performance.timing.navigationStart,
                    page_location: window.location.href
                });
                scrolled = true;
            }
        });
    }

    // Track file downloads
    function trackFileDownload() {
        document.addEventListener("click", function (event) {
            let link = event.target.closest("a");
            if (link && link.href && link.download && /(?:pdf|xlsx?|docx?|txt|rtf|csv|exe|key|pp(s|t|tx)|7z|pkg|rar|gz|zip|avi|mov|mp4|mpe?g|wmv|midi?|mp3|wav|wma)$/i.test(link.href)) {
                sendEvent("file_download", {
                    file_extension: link.href.split('.').pop(),
                    file_name: link.href.split("/").pop(),
                    link_classes: link.className,
                    link_id: link.id || null,
                    link_text: link.textContent.trim(),
                    link_url: link.href
                });
            }
        });
    }

    // Track form starts and submissions
    function trackForms() {
        document.addEventListener("focusin", function (event) {
            if (event.target.closest("form")) {
                sendEvent("form_start", {
                    form_id: event.target.closest("form").id,
                    form_name: event.target.closest("form").name,
                    form_destination: event.target.closest("form").action
                });
            }
        });

        document.addEventListener("submit", function (event) {
            sendEvent("form_submit", {
                form_id: event.target.id,
                form_name: event.target.name,
                form_destination: event.target.action,
                form_submit_text: event.submitter ? event.submitter.value : null
            });
        });
    }

    // Track HTML5 video events
    function trackHTML5Videos() {
        document.querySelectorAll("video").forEach((video) => {
            video.addEventListener("play", () => sendEvent("video_start", {
                video_provider: "HTML5",
                video_title: video.title,
                video_url: video.currentSrc,
                video_current_time: video.currentTime,
                video_duration: video.duration,
                visible: isElementInViewport(video) // Dynamically check if video is visible in viewport
            }));
            video.addEventListener("ended", () => sendEvent("video_complete", {
                video_provider: "HTML5",
                video_title: video.title,
                video_url: video.currentSrc,
                video_duration: video.duration,
                visible: isElementInViewport(video)
            }));
            video.addEventListener("timeupdate", () => {
                let percent = (video.currentTime / video.duration) * 100;
                if ([10, 25, 50, 75].includes(Math.round(percent))) {
                    sendEvent("video_progress", {
                        video_provider: "HTML5",
                        video_title: video.title,
                        video_url: video.currentSrc,
                        video_current_time: video.currentTime,
                        video_duration: video.duration,
                        video_percent: Math.round(percent),
                        visible: isElementInViewport(video)
                    });
                }
            });
        });
    }

    // Track YouTube videos (via embedded iframe)
    function trackYouTubeVideos() {
        if (!window.YT || !YT.Player) return;

        document.querySelectorAll("iframe[src*='youtube.com']").forEach((iframe) => {
            let player = new YT.Player(iframe, {
                events: {
                    onStateChange: function (event) {
                        if (event.data == YT.PlayerState.PLAYING) sendEvent("video_start", {
                            video_provider: "YouTube",
                            video_url: iframe.src,
                            video_title: iframe.title,
                            visible: isElementInViewport(iframe)
                        });
                        if (event.data == YT.PlayerState.ENDED) sendEvent("video_complete", {
                            video_provider: "YouTube",
                            video_url: iframe.src,
                            video_title: iframe.title,
                            visible: isElementInViewport(iframe)
                        });
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

    // Check if an element is in the viewport (for video visibility tracking)
    function isElementInViewport(el) {
        let rect = el.getBoundingClientRect();
        return rect.top >= 0 && rect.left >= 0 && rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) && rect.right <= (window.innerWidth || document.documentElement.clientWidth);
    }

    // ========== FUNCTION CALLS ==========
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
