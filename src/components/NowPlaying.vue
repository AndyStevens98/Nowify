<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
        <h3 style="opacity: 0.8;" v-text="player.trackAlbum.title"></h3>
        <br /><br /><br /><br />
        <h1 style="opacity: 0.5;">Up next:</h1>
        <h2 class="up-next__track" v-text="player.upNextTrackTitle"></h2>
        <h2 class="up-next__artists" v-text="getUpNextTrackArtists"></h2>
        <h2 class="up-next__album" v-text="player.upNextTrackAlbum.title"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      upNextResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },
    getUpNextTrackArtists() {
      return this.player.upNextTrackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}
      let upNextData = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        const upNextResponse = await fetch(
          `${this.endpoints.base}/${this.endpoints.queue}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }
        if (!upNextResponse.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()
        upNextData = await upNextResponse.json()
        this.playerResponse = data
        this.upNextResponse = upNextData
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      /**
       * No image (rare).
       */
      if (!this.player.trackAlbum?.image) {
        return
      }

      /**
       * Run node-vibrant to get colours.
       */
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: '',
        upNextTrackArtists: [],
        upNextTrackTitle: '',
        upNextTrackAlbum: []
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        },
        upNextTrackArtists: this.upNextResponse.queue[0].artists.map(
          artist => artist.name
        ),
        upNextTrackTitle: this.upNextResponse.queue[0].name,
        upNextTrackAlbum: {
          title: this.upNextResponse.queue[0].album.name,
          image: this.upNextResponse.queue[0].album.images[0].url
        }
      }

      /**
       * Store the up next track.
       */
      this.upNextData = {
        trackArtists: this.upNextResponse.queue[0].artists.map(
          artist => artist.name
        ),
        trackTitle: this.upNextResponse.queue[0].name,
        trackId: this.upNextResponse.queue[0].id,
        trackAlbum: {
          title: this.upNextResponse.queue[0].album.name,
          image: this.upNextResponse.queue[0].album.images[0].url
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours

      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },
  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
