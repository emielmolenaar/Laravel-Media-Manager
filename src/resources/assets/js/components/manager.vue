<style lang="scss" scoped src="../../sass/packages/bulma.scss"></style>

<script>
import debounce from 'lodash/debounce'
import Vue2Filters from 'vue2-filters'

import Broadcast from '../modules/broadcast'
import BulkSelect from '../modules/bulk'
import Cache from '../modules/cache'
import Computed from '../modules/computed'
import Download from '../modules/download'
import Form from '../modules/form'
import Folder from '../modules/folder'
import Gestures from '../modules/gestures'
import Image from '../modules/image'
import ItemFiltration from '../modules/filtration'
import ItemVisibility from '../modules/visibility'
import LockItem from '../modules/lock'
import MediaPlayer from '../modules/media-player'
import Restriction from '../modules/restriction'
import Scroll from '../modules/scroll'
import Selection from '../modules/selection'
import Url from '../modules/url'
import Upload from '../modules/upload'
import Utilities from '../modules/utils'
import Watchers from '../modules/watch'

export default {
    components: {
        contentRatio: require('./utils/ratio.vue').default,
        globalSearchBtn: require('./globalSearch/button.vue').default,
        globalSearchPanel: require('./globalSearch/panel.vue').default,
        imageCache: require('./lazyLoading/cache.vue').default,
        imageEditor: require('./imageEditor/main.vue').default,
        imageIntersect: require('./lazyLoading/normal.vue').default,
        overlay: require('./utils/overlay.vue').default,
        usageIntroBtn: require('./usageIntro/button.vue').default,
        usageIntroPanel: require('./usageIntro/panel.vue').default,
        videoDimension: require('./utils/video-dim.vue').default
    },
    name: 'media-manager',
    mixins: [
        Vue2Filters.mixin,
        Broadcast,
        BulkSelect,
        Cache,
        Computed,
        Download,
        Form,
        Folder,
        Gestures,
        Image,
        ItemFiltration,
        ItemVisibility,
        LockItem,
        MediaPlayer,
        Restriction,
        Scroll,
        Selection,
        Url,
        Upload,
        Utilities,
        Watchers
    ],
    props: [
        'config',
        'routes',
        'inModal',
        'translations',
        'uploadPanelImgList',
        'hideExt',
        'hidePath',
        'restrict',
        'userId'
    ],
    data() {
        return {
            ajax_error: false,
            bulkSelect: false,
            bulkSelectAll: false,
            checkForFolders: false,
            disableShortCuts: false,
            firstMeta: false, // for alt + click selection
            firstRun: false, // deffer running logic on init
            folderWarning: false,
            imageWasEdited: false,
            infoSidebar: false,
            introIsOn: false,
            isLoading: false,
            linkCopied: false,
            loading_files: false,
            no_files: false,
            no_search: false,
            randomNames: false,
            showProgress: false,
            smallScreen: false,
            toolBar: true,
            UploadArea: false,
            waitingForUpload: false,
            useCopy: false,

            activeModal: null,
            currentFileIndex: null,
            currentFilterName: null,
            imageSlideDirection: null,
            moveToPath: null,
            newFilename: null,
            newFolderName: null,
            searchFor: null,
            searchItemsCount: null,
            selectedFile: null,
            sortBy: null,
            urlToUpload: null,

            bulkList: [],
            dimensions: [],
            directories: [],
            files: [],
            filterdList: [],
            folders: [],
            selectedUploadPreviewList: [],
            selectedUploadPreviewExistList: [],
            selectedUploadPreview: {
                img: null,
                type: null,
                name: null
            },
            player: {
                item: null,
                fs: false,
                playing: false
            },
            scrollableBtn: {
                state: false,
                dir: 'down'
            },
            lockedList: [],
            uploadPanelGradients: [
                'linear-gradient(141deg, #009e6c 0, #00d1b2 71%, #00e7eb 100%)',
                'linear-gradient(141deg, #04a6d7 0, #209cee 71%, #3287f5 100%)',
                'linear-gradient(141deg, #12af2f 0, #23d160 71%, #2ce28a 100%)',
                'linear-gradient(141deg, #ffaf24 0, #ffdd57 71%, #fffa70 100%)',
                'linear-gradient(141deg, #ff0561 0, #ff3860 71%, #ff5257 100%)',
                'linear-gradient(141deg, #1f191a 0, #363636 71%, #46403f 100%)'
            ],
            progressCounter: 0,
            scrollByRows: 0
        }
    },
    created() {
        // confirm file pending upload cancel
        window.addEventListener('beforeunload', (event) => {
            if (this.showProgress) {
                event.preventDefault()
                event.returnValue = 'Current Upload Will Be Canceled !!'
            }
        })
        window.addEventListener('unload', (event) => {
            if (this.showProgress) {
                EventHub.fire('clear-pending-upload')
            }
        })

        // rest
        window.addEventListener('resize', this.onResize)
        window.addEventListener('popstate', this.urlNavigation)
        document.addEventListener('keydown', this.shortCuts)
        this.init()
    },
    mounted() {
        this.eventsListener()
        this.scrollObserve()
    },
    updated: debounce(function() {
        if (this.firstRun) {
            this.updateScrollByRow()

            if (this.selectedFileIs('video') || this.selectedFileIs('audio')) {
                this.destroyPlyr()
                this.$nextTick(this.initPlyr)
            }

            if (!this.introIsOn) {
                this.activeModal || this.inModal
                    ? this.noScroll('add')
                    : this.noScroll('remove')
            }
        }
    }, 250),
    beforeDestroy() {
        window.removeEventListener('resize', this.onResize)
        window.removeEventListener('popstate', this.urlNavigation)
        document.removeEventListener('keydown', this.shortCuts)
        this.destroyPlyr()
        this.noScroll('remove')
    },
    methods: {
        init() {
            // restricted
            if (this.restrictModeIsOn()) {
                this.clearUrlQuery()
                this.resolveRestrictFolders()

                return this.getFiles(this.folders).then(() => {
                    this.fileUpload()
                    this.$nextTick(() => {
                        this.firstRun = true
                        this.onResize()
                    })
                })
            }

            // normal
            this.getPathFromUrl()
                .then(this.preSaved())
                .then(() => {
                    return this.getFiles(this.folders, null, this.selectedFile).then(() => {
                        this.fileUpload()
                        this.$nextTick(() => {
                            this.firstRun = true
                            this.onResize()
                        })
                    })
                })
        },

        eventsListener() {
            // check if image was edited
            EventHub.listen('image-edited', (msg) => {
                this.imageWasEdited = true
                this.removeCachedResponse().then(() => {
                    this.showNotif(`${this.trans('save_success')} "${msg}"`)
                })
            })

            // get images dimensions
            EventHub.listen('save-image-dimensions', (obj) => {
                let dim = this.dimensions

                if (!dim.some((e) => e.url === obj.url)) {
                    dim.push(obj)
                }
            })

            // stop listening to shortcuts
            EventHub.listen('disable-global-keys', (val) => {
                this.disableShortCuts = val
            })

            // gls
            EventHub.listen('search-go-to-folder', (data) => {
                this.folders = this.arrayFilter(data.dir.split('/'))

                return this.getFiles(this.folders, null, data.name).then(() => {
                    this.updatePageUrl()
                })
            })
        },

        shortCuts(e) {
            let key = keycode(e)

            if (!(this.isLoading || e.altKey || e.ctrlKey || e.metaKey || this.disableShortCuts)) {
                // when modal isnt visible
                if (!this.activeModal && !this.waitingForUpload) {
                    // when search is not focused
                    if (!this.isFocused('search', e)) {
                        // when no bulk selecting
                        if (!this.isBulkSelecting()) {
                            // open folder
                            if (key == 'enter' && this.selectedFile) {
                                this.openFolder(this.selectedFile)
                            }

                            // go up a dir
                            if (key == 'backspace' && this.folders.length) {
                                e.preventDefault()
                                this.goToPrevFolder()
                            }

                            // when there are files
                            if (this.allItemsCount) {
                                this.navigation(e)

                                if (key == 'space' &&
                                    e.target == document.body &&
                                    (
                                        this.selectedFileIs('video') ||
                                        this.selectedFileIs('audio') ||
                                        this.selectedFileIs('image') ||
                                        this.selectedFileIs('pdf') ||
                                        this.textFileType()
                                    )
                                ) {
                                    e.preventDefault()

                                    // play-pause media
                                    if (this.selectedFileIs('video') || this.selectedFileIs('audio')) {
                                        this.infoSidebar
                                            ? this.playMedia()
                                            : this.toggleModal('preview_modal')
                                    }

                                    // "show" image/pdf/text quick view
                                    if (this.selectedFileIs('image') || this.selectedFileIs('pdf') || this.textFileType()) {
                                        this.toggleModal('preview_modal')
                                    }
                                }

                                // image editor
                                if (key == 'e') {
                                    this.$refs.editor.click()
                                }
                            }
                            // end of when there are files

                            // refresh
                            if (key == 'r') {
                                if (!this.$refs.refresh.$el.disabled && !this.isLoading) {
                                    this.refresh()
                                }
                            }

                            // file upload
                            if (key == 'u') {
                                this.$refs.upload.click()
                            }

                            if (key == 'esc') {
                                // hide upload panel
                                if (this.UploadArea) {
                                    this.toggleUploadPanel()
                                }

                                // clear filter
                                if (this.currentFilterName) {
                                    this.showFilesOfType('all')
                                }
                            }
                        }
                        // end of no bulk selection

                        // we have files
                        if (this.allItemsCount) {
                            // bulk select
                            if (key == 'b') {
                                if (this.searchFor && this.searchItemsCount == 0) {
                                    return
                                }

                                this.$refs.bulkSelect.click()
                            }

                            if (this.isBulkSelecting()) {
                                // add all to bulk list
                                if (key == 'a') {
                                    this.$refs.bulkSelectAll.click()
                                }

                                // cancel bulk selection
                                if (key == 'esc') {
                                    this.$refs.bulkSelect.click()
                                }
                            }

                            // delete file
                            if (key == 'delete' || key == 'd') {
                                this.$refs.delete.click()
                            }

                            // move file
                            if (this.checkForFolders && key == 'm') {
                                this.$refs.move.click()
                            }

                            // lock files
                            if (key == 'l') {
                                this.$refs.lock.click()
                            }

                            // set visibility
                            if (key == 'v') {
                                this.$refs.vis.click()
                            }
                        }
                        // end of we have files

                        // toggle file details sidebar
                        if (key == 't' && !this.smallScreen) {
                            this.toggleInfoSidebar()
                            this.saveUserPref()
                        }
                    }
                    // end of search is not focused

                    // cancel search
                    else if (key == 'esc') {
                        this.resetInput('searchFor')
                    }
                }
                // end of modal isnt visible

                // when modal is visible
                else {
                    if (this.isActiveModal('preview_modal')) {
                        if (key == 'space') {
                            this.selectedFileIs('video') || this.selectedFileIs('audio')
                                ? this.playMedia()
                                : this.toggleModal()
                        }

                        this.navigation(e)
                    }

                    // hide modal
                    if (key == 'esc' && !this.isActiveModal('imageEditor_modal')) {
                        this.toggleModal()
                    }
                }
                // end of modal is visible

                // when upload preview is visible
                if (this.waitingForUpload) {
                    // proceed with upload
                    if (key == 'enter') {
                        this.$refs['process-dropzone'].click()
                    }

                    // clear upload queue
                    if (key == 'esc') {
                        if (this.UploadArea) {
                            return this.toggleUploadPanel()
                        }

                        this.$refs['clear-dropzone'].click()
                        this.waitingForUpload = false
                    }

                    // trigger upload panel
                    if (key == 'u') {
                        this.$refs.upload.click()
                    }
                }
            }
        },
        // end of short cuts

        refresh() {
            EventHub.fire('clear-global-search')
            this.resetInput('searchFor')

            return this.getFiles(
                this.folders,
                null,
                this.selectedFile ? this.selectedFile.name : null
            )
        },
        clearAll() {
            if (!this.isLoading) {
                this.clearUrlQuery()
                this.clearLs()
                this.clearCache()
                this.clearImageCache()
                this.ajaxError(false)
            }
        },
        moveItem() {
            this.$nextTick(() => {
                if (this.$refs.move.disabled) {
                    return
                }

                this.toggleModal('move_file_modal')
            })
        },
        renameItem() {
            this.$nextTick(() => {
                if (this.$refs.rename.disabled) {
                    return
                }

                this.toggleModal('rename_file_modal')
            })
        },
        deleteItem() {
            this.$nextTick(() => {
                if (this.$refs.delete.disabled) {
                    return
                }

                if (!this.isBulkSelecting() && this.selectedFile) {
                    this.selectedFileIs('folder')
                        ? this.folderWarning = true
                        : this.folderWarning = false
                }

                if (this.bulkItemsCount) {
                    this.bulkItemsFilter.some((item) => {
                        return this.fileTypeIs(item, 'folder')
                            ? this.folderWarning = true
                            : this.folderWarning = false
                    })
                }

                this.toggleModal('confirm_delete_modal')
            })
        },
        createNewFolder() {
            this.toggleModal('new_folder_modal')
        }
    },
    render() {}
}
</script>
