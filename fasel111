package com.lagradost.cloudstream3.movieproviders

import com.lagradost.cloudstream3.*
import com.lagradost.cloudstream3.utils.*
import org.jsoup.nodes.Element

class FaselHDProvider : MainAPI() {
    override var mainUrl = "https://www.faselhds.care"
    override var name = "FaselHD"
    override val hasMainPage = true
    override var lang = "ar"
    override val hasDownloadSupport = true
    override val supportedTypes = setOf(
        TvType.Movie,
        TvType.TvSeries,
        TvType.Anime
    )

    override val mainPage = mainPageOf(
        "$mainUrl/movies" to "أفلام",
        "$mainUrl/series" to "مسلسلات", 
        "$mainUrl/anime" to "أنمي"
    )

    override suspend fun getMainPage(page: Int, request: MainPageRequest): HomePageResponse {
        val document = app.get(request.data + "?page=$page").document
        val home = document.select("div.movie-item").mapNotNull {
            it.toSearchResult()
        }
        return newHomePageResponse(request.name, home)
    }

    private fun Element.toSearchResult(): SearchResponse? {
        val title = this.selectFirst("h3 a")?.text()?.trim() ?: return null
        val href = fixUrl(this.selectFirst("h3 a")?.attr("href") ?: return null)
        val posterUrl = fixUrlNull(this.selectFirst("img")?.attr("src"))
        
        return newMovieSearchResponse(title, href, TvType.Movie) {
            this.posterUrl = posterUrl
        }
    }

    override suspend fun search(query: String): List<SearchResponse> {
        val searchResponse = app.get("$mainUrl/search?q=$query").document
        return searchResponse.select("div.movie-item").mapNotNull {
            it.toSearchResult()
        }
    }

    override suspend fun load(url: String): LoadResponse? {
        val document = app.get(url).document
        
        val title = document.selectFirst("h1")?.text()?.trim() ?: return null
        val poster = fixUrlNull(document.selectFirst("img.poster")?.attr("src"))
        val description = document.selectFirst("div.description")?.text()?.trim()
        val year = document.selectFirst("span.year")?.text()?.toIntOrNull()
        val rating = document.selectFirst("span.rating")?.text()?.toDoubleOrNull()
        
        // Check if it's a series or movie
        val episodes = document.select("div.episode-item")
        
        return if (episodes.isNotEmpty()) {
            // It's a series
            val episodeList = episodes.mapNotNull { ep ->
                val episodeTitle = ep.selectFirst("span.episode-title")?.text() ?: return@mapNotNull null
                val episodeHref = fixUrl(ep.selectFirst("a")?.attr("href") ?: return@mapNotNull null)
                val episodeNum = ep.selectFirst("span.episode-number")?.text()?.toIntOrNull()
                
                Episode(
                    data = episodeHref,
                    name = episodeTitle,
                    episode = episodeNum
                )
            }
            
            newTvSeriesLoadResponse(title, url, TvType.TvSeries, episodeList) {
                this.posterUrl = poster
                this.plot = description
                this.year = year
                this.rating = rating?.times(1000)?.toInt()
            }
        } else {
            // It's a movie
            newMovieLoadResponse(title, url, TvType.Movie, url) {
                this.posterUrl = poster
                this.plot = description
                this.year = year
                this.rating = rating?.times(1000)?.toInt()
            }
        }
    }

    override suspend fun loadLinks(
        data: String,
        isCasting: Boolean,
        subtitleCallback: (SubtitleFile) -> Unit,
        callback: (ExtractorLink) -> Unit
    ): Boolean {
        val document = app.get(data).document
        
        // Look for video sources
        document.select("div.server-item").forEach { server ->
            val serverName = server.selectFirst("span.server-name")?.text() ?: "Unknown"
            val serverUrl = server.selectFirst("a")?.attr("href") ?: return@forEach
            
            // Extract video URL from the server page
            val videoDoc = app.get(fixUrl(serverUrl)).document
            val videoUrl = videoDoc.selectFirst("iframe")?.attr("src") 
                ?: videoDoc.selectFirst("video source")?.attr("src")
            
            if (!videoUrl.isNullOrEmpty()) {
                callback.invoke(
                    ExtractorLink(
                        name,
                        serverName,
                        fixUrl(videoUrl),
                        referer = data,
                        quality = Qualities.Unknown.value,
                        isM3u8 = videoUrl.contains(".m3u8")
                    )
                )
            }
        }
        
        return true
    }
}