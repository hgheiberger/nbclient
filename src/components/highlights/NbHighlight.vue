<template>
    <g
        class="nb-highlight"
        v-if="visible"
        :style="style"
        v-tooltip="{
            content: getTooltipContent(),
        }"
    >
        <rect
            v-for="(box, index) in bounds.boxes"
            :key="index"
            :x="box.left + bounds.offsetX"
            :y="box.top + bounds.offsetY"
            :height="box.height"
            :width="box.width"
            visibility="hidden">
            <animate
                v-if="showRecentActivityAnimation"
                attributeType="XML"
                attributeName="fill"
                values="#ffffff;#4a2270D9;#ffffff;#ffffff"
                dur="5.0s"
                repeatCount="indefinite"/>
            <animate
                v-if="showTypingActivityAnimation"
                attributeType="XML"
                attributeName="fill"
                values="#ffffff;#4a2270D9;#ffffff;#ffffff"
                dur="2.0s"
                repeatCount="indefinite"/>
        </rect>

    </g>
    <g
        class="nb-highlight"
        v-else-if="isHidden"
        :style="style"
        v-tooltip.right="{content: getHiddenTooltipContent()}"
        @click="$emit('select-thread',thread)"
        @mouseenter="onHover(true)"
        @mouseleave="onHover(false)">
        <rect
            v-for="(box, index) in bounds.boxes"
            :key="index"
            :x="bounds.offsetX"
            :y="box.top + bounds.offsetY"
            :height="box.height"
            width="20">
        </rect>
    </g>
    <g
        class="nb-highlight"
        v-else
        :style="style"
        @click="$emit('select-thread',thread)"
        @mouseenter="onHover(true)"
        @mouseleave="onHover(false)">
        <rect
            v-for="(box, index) in bounds.boxes"
            :key="index"
            :x="bounds.offsetX"
            :y="box.top + bounds.offsetY"
            :height="box.height"
            width="20">
        </rect>
    </g>
</template>

<script>
import { getTextBoundingBoxes } from '../../utils/overlay-util.js'
import axios from 'axios'

/**
 * Component for individual highlight overlay corresponding to each thread.
 * Each thread is represented by the head of thread {@link NbComment}.
 *
 * @vue-prop {?NbComment} thread - thread for this highlight,
 *   null if this is a draft
 * @vue-prop {NbComment} threadSelected - currently selected thread
 * @vue-prop {Array<NbComment>} threadsHovered=[] - currently hovered threads
 * @vue-prop {NbRange} range - text range for this higlight
 * @vue-prop {Rect} drawAnnotationDraftRect - HTML rect element for the draw annotation draft
 * @vue-prop {SVG} drawAnnotationDraftSvg - SVG element in which to insert the draft rect
 * @vue-prop {Boolean} showHighlights=true - true if highlights are overlayed
 *   on text, false if collapsed to the side
 *
 * @vue-computed {String} style - additional CSS for highlight color
 *   in case this is a draft, selected, or hovered
 * @vue-computed {Object} bounds - bounds of highlight
 * @vue-computed {Array<Rect>} bounds.boxes - bouding box rectangles of the
 *   text range, calculated by {@link getTextBoundingBoxes}
 * @vue-computed {Number} bounds.offsetX - x coordinate offset of highlight
 * @vue-computed {Number} bounds.offsetY - y coordinate offset of highlight
 * @vue-computed {Boolan} visible - true if this highlight should be overlayed
 *   on text (i.e. showHighlights is true or this thread is selected)
 *
 * @vue-event {NbComment} select-thread - Emit this thread when user clicks on
 *   this highlight
 * @vue-event {NbComment} hover-thread - Emit this thread when user starts
 *   hovering over this highlight
 * @vue-event {NbComment} unhover-thread - Emit this thread when user stops
 *   hovering over this highlight
 */
export default {
    name: 'nb-highlight',
    props: {
        thread: Object,
        user: Object,
        threadSelected: Object,
        threadsHovered: {
            type: Array,
            default: () => []
        },
        range: Object,
        drawAnnotationDraftRect: Object,
        drawAnnotationDraftSvg: Object,
        showHighlights: {
            type: Boolean,
            default: true
        },
        showSpotlights: {
            type: Boolean,
            default: true
        },
        activeClass: {
            type: Object,
            default: () => {}
        },
        showSyncFeatures: {
            type: Boolean,
            default: false
        }, 
        isHidden: {
            type: Boolean,
            default: false
        }, 
        currentConfigs: {
            type: Object,
            default: () => {}
        },
    },
    data () {
        return {
        recent: false,
        isHovered: false,
        }
    },
    mounted () {
        // Override css stylesheet of pdf_viewer library to fix CSS Custom Highlights styles
        if (window.location.pathname === '/nb_viewer.html') {
            const style = document.createElement('style')
            style.innerHTML = `
                .textLayer {
                    opacity: 1 !important;
                }
            `
            document.head.appendChild(style)
        }

        if (this.thread) {
            const totalTime = 60000
            let inView = true
            let rect = this.$el.getBoundingClientRect()
            let elTop = rect.top
            let elHeight = rect.height
            let viewHeight = window.innerHeight
            if ((elTop + elHeight) > viewHeight) { // past the user's location (true if before the user location)
                inView = false
            }
            let timeDiff = Date.now() - this.thread.getMostRecentTimeStamp()
            if (timeDiff < totalTime) {
                this.recent = true // show blue highlighting for any comments that were recently posted ~
                if (inView) {
                    this.$emit('new-recent-thread', this.thread) // emit to display notification if in view of current user
                } 
                setTimeout(() => {
                    this.recent = false // set back to false after the time diff is over
                }, totalTime-timeDiff) // we still have 60 seconds - time diff left to display this recent annotation
            }
        }
        document.addEventListener('mousemove', this.handleMouseMove)
        document.addEventListener('click', this.handleMouseClick)
        this.generateHighlights()
    },
    unmounted () {
        let oldAnnotation = document.getElementById(this.highlightId)
        if (oldAnnotation) {
            oldAnnotation.remove()
        }

        document.removeEventListener('mousemove', this.handleMouseMove)
        document.removeEventListener('click', this.handleMouseClick)
        CSS.highlights.delete(this.highlightId)
        const existingStyle = document.querySelector(`style[highlight-id="${this.highlightId}"]`)
        if (existingStyle) {
            existingStyle.remove()
        }
    },
    watch: {
        /**
        * When the currently selected thread changes, check if the highlight is
        * in the view. If not, scroll down/up the window to center the highlight.
        */
        threadSelected: function (val) {
            if (this.thread !== val) { return }

            let nodeContainingRange
            if (this.thread && this.thread.drawAnnotationRect) {
                nodeContainingRange = this.thread.drawAnnotationSvg
            } else {
                nodeContainingRange = this.thread.range.toRange().commonAncestorContainer
            }

            let rect = nodeContainingRange.getBoundingClientRect()
            let elTop = rect.top
            let elHeight = rect.height
            let viewHeight = window.innerHeight
            if (elTop < 0 || (elTop + elHeight) > viewHeight) {
                nodeContainingRange.scrollIntoView({ behavior: 'smooth', block: 'center' })
            }
        },
        visible: function (val) {
            this.generateHighlights()
        },
        thread: function (val) {
            this.generateHighlights()
        },
        range: function (val) {
            this.generateHighlights()
        },
        drawAnnotationDraftRect: function (val) {
            this.generateHighlights()
        },
        highlightId: function (newVal, oldVal) {
            let oldAnnotation = document.getElementById(oldVal)
            if (oldAnnotation) {
                oldAnnotation.remove()
            }
            CSS.highlights.delete(oldVal)
            this.generateHighlights()

            const existingStyle = document.querySelector(`style[highlight-id="${oldVal}"]`)
            if (existingStyle) {
                existingStyle.remove()
            }
        },
        style: function (val) {
            this.updateHighlightStyle()
        }
    },
    computed: {
        spotlight: function () {
            return this.thread.systemSpotlight ? this.thread.systemSpotlight : this.thread.spotlight
        },
        style: function () {
            if (this.isHidden) {
                return "fill: none; stroke: rgb(255 204 1 / 95%); stroke-dasharray: 3;"
            }
            if (!this.thread) {
                return 'fill: rgb(231, 76, 60); fill-opacity: 0.3; cursor: pointer;'
            }
            if (this.thread === this.threadSelected) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.3; cursor: pointer;'
            }
            if (this.threadsHovered.includes(this.thread)) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.12; cursor: pointer;'
            }
            if (this.showSpotlights && this.spotlight && this.spotlight.type === 'EM' && this.currentConfigs.isEmphasize) {
                let color = this.spotlight.color? this.spotlight.color : 'lime'
                return `stroke: ${color}; fill: ${color}; fill-opacity: 0.3; stroke-opacity: 0.9; stroke-dasharray: 1,1; stroke-width: 2px; cursor: pointer;`
            }
            if (this.showTypingActivityAnimation) { // if typing, show a pink outline color
                // return 'stroke: rgb(255, 0, 255); stroke-width: 25'
                return
            }
            // if (this.showRecentActivityAnimation) { // if recently shown, show a cyan outline color
            //     // return 'stroke: rgb(0, 255, 255); stroke-width: 15'
            //     return
            // }
            // if (this.unseenNotificationThread) {
            //     return 'fill: rgb(80, 54, 255); opacity: 0.7;'
            //     // return 'stroke: rgb(80, 54, 255); stroke-width: 8; stroke-opacity: 0.2;'
            // }
            // if (this.replyRequestThread) {
            //     if (this.thread.isUnseen() && this.currentConfigs.isShowIndicatorForUnseenThread) {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.25;'
            //         return 'fill: rgb(255, 0, 255); opacity: 1.0;'
            //     } else {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.10;'
            //         return 'fill: rgb(255, 0, 255); opacity: 0.5;'
            //     }
            // }
            return 'fill: rgb(255, 204, 1); opacity: 0.2; cursor: pointer;'
        },
        drawAnnotationStyle: function () {
            if (this.isHidden) {
                return "fill: none; stroke: rgb(255 204 1 / 95%); stroke-dasharray: 3;"
            }
            if (!this.thread) {
                return 'fill: rgb(231, 76, 60); fill-opacity: 0.3; cursor: pointer; stroke: rgb(67, 14, 8); stroke-opacity: 0.9; stroke-width: 3px;'
            }
            if (this.thread === this.threadSelected) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.35; cursor: pointer; stroke: rgb(0, 15, 40); stroke-opacity: 0.9; stroke-width: 3px;'
            }
            if (this.threadsHovered.includes(this.thread)) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.18; cursor: pointer; stroke: rgb(0, 23, 60); stroke-opacity: 0.9; stroke-width: 3px;'
            }
            if (this.showSpotlights && this.spotlight && this.spotlight.type === 'EM' && this.currentConfigs.isEmphasize) {
                let color = this.spotlight.color? this.spotlight.color : 'lime'
                return `stroke: ${color}; fill: ${color}; fill-opacity: 0.3; stroke-opacity: 0.9; stroke-dasharray: 1,1; stroke-width: 3px; cursor: pointer;`
            }
            if (this.showTypingActivityAnimation) { // if typing, show a pink outline color
                // return 'stroke: rgb(255, 0, 255); stroke-width: 25'
                return
            }
            // if (this.showRecentActivityAnimation) { // if recently shown, show a cyan outline color
            //     // return 'stroke: rgb(0, 255, 255); stroke-width: 15'
            //     return
            // }
            // if (this.unseenNotificationThread) {
            //     return 'fill: rgb(80, 54, 255); opacity: 0.7;'
            //     // return 'stroke: rgb(80, 54, 255); stroke-width: 8; stroke-opacity: 0.2;'
            // }
            // if (this.replyRequestThread) {
            //     if (this.thread.isUnseen() && this.currentConfigs.isShowIndicatorForUnseenThread) {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.25;'
            //         return 'fill: rgb(255, 0, 255); opacity: 1.0;'
            //     } else {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.10;'
            //         return 'fill: rgb(255, 0, 255); opacity: 0.5;'
            //     }
            // }
            return 'fill: rgb(255, 204, 1); opacity: 0.35; cursor: pointer; stroke: rgb(40, 32, 0); stroke-opacity: 0.9; stroke-width: 3px;'
        },
        highlightStyle: function () {
            if (this.isHidden) {
                return "background-color: none; stroke: rgb(255 204 1 / 95%); stroke-dasharray: 3;"
            }
            if (!this.thread) {
                return 'background-color: rgba(231, 76, 60, 0.3); cursor: pointer;'
            }
            if (this.thread === this.threadSelected) {
                return 'background-color: rgba(1, 99, 255, 0.3);'
            }
            if (this.threadsHovered.includes(this.thread)) {
                return 'background-color: rgba(1, 99, 255, 0.12); fill-opacity: 0.12;'
            }
            if (this.showSpotlights && this.spotlight && this.spotlight.type === 'EM' && this.currentConfigs.isEmphasize) {
                let color = this.spotlight.color? this.spotlight.color : 'rgba(0, 255, 0, 0.3)'
                return `stroke: ${color}; background-color: ${color}; stroke-opacity: 0.9; stroke-dasharray: 1,1; stroke-width: 2px;`
            }
            if (this.showTypingActivityAnimation) { // if typing, show a pink outline color
                // return 'stroke: rgb(255, 0, 255); stroke-width: 25'
                return
            }
            // if (this.showRecentActivityAnimation) { // if recently shown, show a cyan outline color
            //     // return 'stroke: rgb(0, 255, 255); stroke-width: 15'
            //     return
            // }
            // if (this.unseenNotificationThread) {
            //     return 'fill: rgb(80, 54, 255); opacity: 0.7;'
            //     // return 'stroke: rgb(80, 54, 255); stroke-width: 8; stroke-opacity: 0.2;'
            // }
            // if (this.replyRequestThread) {
            //     if (this.thread.isUnseen() && this.currentConfigs.isShowIndicatorForUnseenThread) {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.25;'
            //         return 'fill: rgb(255, 0, 255); opacity: 1.0;'
            //     } else {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.10;'
            //         return 'fill: rgb(255, 0, 255); opacity: 0.5;'
            //     }
            // }
            return 'background-color: rgba(255, 204, 1, 0.2);'
        },
        isRecentThread: function () {
            return this.thread && this.recent && this.showSyncFeatures
        },
        isTypingThread: function () {
            return this.thread && this.thread.usersTyping && this.thread.usersTyping.length > 0 && this.showSyncFeatures
        },
        showRecentActivityAnimation: function () {
            return false
            // if (this.thread && ( (this.thread === this.threadSelected) || this.threadsHovered.includes(this.thread))) { // if typing or hover, don't animate
            //     return false
            // }
            // return this.thread && this.recent && this.showSyncFeatures
        },
        showTypingActivityAnimation: function () {
            return false
            // if (this.thread && ( (this.thread === this.threadSelected) || this.threadsHovered.includes(this.thread))) { // if typing or hover, don't animate
            //     return false
            // }
            // return this.thread && this.thread.usersTyping && this.thread.usersTyping.length > 0 && this.showSyncFeatures
        },
        replyRequestThread: function () {
            return this.showSyncFeatures && this.thread && this.thread.hasReplyRequests()
        },
        unseenNotificationThread: function () {
            return this.thread && this.thread.associatedNotification !== null && this.showSyncFeatures && this.thread.associatedNotification.unseen
        },
        bounds: function () {
            let bounds = {}
            if (this.drawAnnotationDraftRect || (this.thread && this.thread.drawAnnotationRect)) {
                bounds.boxes = {}
                return bounds
            } else if (this.thread) {
                bounds.boxes = getTextBoundingBoxes(this.thread.range.toRange())
            } else {
                bounds.boxes = getTextBoundingBoxes(this.range.toRange())
            }
            bounds.offsetX = window.pageXOffset || document.documentElement.scrollLeft || document.body.scrollLeft || 0
            bounds.offsetY = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || 0
            return bounds
        },
        visible: function () {
            return !this.isHidden && (this.showHighlights || (this.thread === this.threadSelected) || (this.showSpotlights && this.spotlight  && this.spotlight.type === 'EM'))
        },
        highlightId: function () {
            if (this.thread && this.thread.drawAnnotationRect) {
                return `id${this.thread.id.substring(0, 12)}`
            } else if (this.drawAnnotationDraftRect) {
                return `id${this.drawAnnotationDraftSvg.getBBox()}-${this.drawAnnotationDraftRect.x}-${this.drawAnnotationDraftRect.y}-${this.drawAnnotationDraftRect.width}-${this.drawAnnotationDraftRect.height}`
            } else if (this.thread) {
                return `id${this.thread.id.substring(0, 12)}`
            } else {
                let range = this.range.toRange()
                return `${range.startContainer.nodeValue}-${range.startOffset}-${range.endOffset}`
            }
        },
    },
    methods: {
        onHover: function (state) {
            this.$emit(state ? 'hover-thread' : 'unhover-thread', this.thread)
        },
        // TODO 2023: Check this
        onClick: function () {
            if (!this.thread) {
                return this.$emit('select-thread', this.thread, 'NONE')
            }

            const initiator = this.currentConfigs.isEmphasize && this.spotlight && this.spotlight.type === 'EM' ? 'SPOTLIGHT' : 'HIGHLIGHT'
            this.$emit('log-nb', 'CLICK', initiator, this.thread)

            let type = 'HIGHLIGHT'
            
            if ((this.currentConfigs.isEmphasize && this.spotlight && this.spotlight.type === 'EM') || (this.currentConfigs.isInnotation && this.spotlight && this.spotlight.type === 'IN')) {
                type = this.spotlight.type.toUpperCase()
            }

            const source = window.location.pathname === '/nb_viewer.html' ? window.location.href : window.location.origin + window.location.pathname
            const token = localStorage.getItem("nb.user");
            const config = { headers: { Authorization: 'Bearer ' + token }, params: { url: source } }

            try {
                axios.post(`/api/spotlights/log`, {
                    spotlight_id: type === 'HIGHLIGHT' || this.thread.systemSpotlight ? null : this.spotlight.id,
                    action: 'CLICK', 
                    type: type, 
                    annotation_id: this.thread.id, 
                    class_id: this.activeClass.id,
                    role: this.user.role.toUpperCase()
                }, config)
            } catch (error) {}

            this.logNbClick()

            if (this.currentConfigs.isEmphasize && this.spotlight && (this.spotlight.type === 'EM' || this.spotlight.type === 'IN')) {
                this.$emit('select-thread', this.thread, 'SPOTLIGHT')
            } else {
                this.$emit('select-thread', this.thread, 'HIGHLIGHT')
            }
        },
        handleMouseMove: function (event) {
            if (this.thread && this.thread.drawAnnotationRect && this.visible) {
                let rect = document.getElementById(this.highlightId)
                let isInside = false
                if (rect) {
                    let bbox = rect.getBoundingClientRect()
                    const mouseX = event.clientX
                    const mouseY = event.clientY
                    isInside = mouseX >= bbox.left && mouseX <= bbox.right && mouseY >= bbox.top && mouseY <= bbox.bottom
                }
                if (isInside) {
                    this.isHovered = true
                    this.onHover(true)
                } else if (this.isHovered) {
                    this.isHovered = false
                    this.onHover(false)
                }
            } else if (this.thread && this.visible) {
                const mousePoint = document.caretPositionFromPoint(event.clientX, event.clientY)
                const range = this.thread.range.toRange()
                const existingStyle = document.querySelector(`style[highlight-id="${this.highlightId}"]`)
                if (existingStyle && mousePoint && range.isPointInRange(mousePoint.offsetNode, mousePoint.offset)) {
                    this.isHovered = true
                    this.onHover(true)
                } else if (this.isHovered) {
                    this.isHovered = false
                    this.onHover(false)
                }
            }
        },
        handleMouseClick: function (event) {
            if (this.thread && this.thread.drawAnnotationRect && this.visible) {
                let rect = document.getElementById(this.highlightId)
                let isInside = false
                if (rect) {
                    let bbox = rect.getBoundingClientRect()
                    const mouseX = event.clientX
                    const mouseY = event.clientY
                    isInside = mouseX >= bbox.left && mouseX <= bbox.right && mouseY >= bbox.top && mouseY <= bbox.bottom
                }
                if (isInside) {
                    this.onClick()
                }
            } else if (this.thread && this.visible) {
                const mousePoint = document.caretPositionFromPoint(event.clientX, event.clientY)
                const range = this.thread.range.toRange()
                const existingStyle = document.querySelector(`style[highlight-id="${this.highlightId}"]`)
                if (existingStyle && mousePoint && range.isPointInRange(mousePoint.offsetNode, mousePoint.offset)) {
                    this.onClick()
                }
            }
        },
        logNbClick: function () {
            if (this.unseenNotificationThread || this.isTypingThread || this.isRecentThread || this.showTypingActivityAnimation) {
                let trigger_type = ''
                if (this.isTypingThread || this.isRecentThread) {
                    trigger_type = 'USER_SAW_RECENT_ACTIVITY'
                } else if (this.unseenNotificationThread) {
                    trigger_type = this.thread.associatedNotification.trigger 
                } else {
                    trigger_type = 'REPLY_REQUESTED'
                }
                // console.log(trigger_type)
                const source = window.location.pathname === '/nb_viewer.html' ? window.location.href : window.location.origin + window.location.pathname
                const token = localStorage.getItem("nb.user");
                const config = { headers: { Authorization: 'Bearer ' + token }, params: { url: source } }

                try {
                    axios.post(`/api/spotlights/log`, {
                        spotlight_id: null,
                        action: 'CLICK', 
                        type: 'NOTIFICATION_HIGHLIGHT', 
                        annotation_id: this.thread.id, 
                        class_id: this.activeClass.id,
                        role: this.user.role.toUpperCase(),
                        trigger: trigger_type
                    }, config)
                } catch (error) {}

            }
        },
        getHiddenTooltipContent: function () {
            let content = "<span>Filtered comment:</span>"
            content += "<br>"
            let text = this.thread.text
            content += text.substring(0, 30)
            if (text.length > 30) {
                content += "..."
            }
            return content

        },
        getTooltipContent: function () {
            if (!this.thread || !this.showSyncFeatures) {
                return ""
            }
            let content = ""
            if (this.isRecentThread || this.isTypingThread) {
                content = "<span>recent comment:</span>"
            } else if (this.unseenNotificationThread) {
                content = "<span>" + 
                this.thread.associatedNotification.readableType + " notification:</span>"
            } else if (this.replyRequestThread) {
                content = "<span>reply requested comment:</span>"
            } else {
                return "" // no associated notification, return empty string
            }
            content += "<br>"

            let relevantComment = 
                (this.unseenNotificationThread && this.thread.associatedNotification.specificAnnotation !== null) 
                ? this.thread.associatedNotification.specificAnnotation : this.thread 

            let text = relevantComment.text
            content += text.substring(0, 30)
            if (text.length > 30) {
                content += "..."
            }
            return content
        },
        updateHighlightStyle: function () {
            // handle draw annotations
            if (this.thread && this.thread.drawAnnotationRect) {
                let annotation = document.getElementById(this.highlightId)
                annotation.setAttributeNS(null, 'style', this.drawAnnotationStyle)
                return
            }

            const existingStyle = document.querySelector(`style[highlight-id="${this.highlightId}"]`)
            if (existingStyle) {
                existingStyle.remove()
            }
            const style = document.createElement('style')
            style.innerHTML = `
                ::highlight(${this.highlightId}) {
                    ${this.highlightStyle}
                }
            `
            style.setAttribute('highlight-id', `${this.highlightId}`)
            document.head.appendChild(style)
        },
        generateHighlights: function () {
            // Handles draw annotations
            if (this.drawAnnotationDraftRect) {
               if (document.body.contains(this.drawAnnotationDraftRect)) {
                this.drawAnnotationDraftRect.remove()
               }
                this.drawAnnotationDraftRect.style = this.drawAnnotationStyle
                this.drawAnnotationDraftSvg.appendChild(this.drawAnnotationDraftRect)
                return
            } else if (this.thread && this.thread.drawAnnotationRect) {
                let oldAnnotation = document.getElementById(this.highlightId)
                if (oldAnnotation) {
                    oldAnnotation.remove()
                }

                let rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect')
                const boundingBox = this.thread.drawAnnotationSvg.getBoundingClientRect()
                rect.setAttributeNS(null, 'x', this.thread.drawAnnotationRect.x_offset * boundingBox.width)
                rect.setAttributeNS(null, 'y', this.thread.drawAnnotationRect.y_offset * boundingBox.height)
                rect.setAttributeNS(null, 'rx', 12)
                rect.setAttributeNS(null, 'width', this.thread.drawAnnotationRect.width * boundingBox.width)
                rect.setAttributeNS(null, 'height', this.thread.drawAnnotationRect.height * boundingBox.height)
                rect.setAttributeNS(null, 'id', this.highlightId)
                rect.setAttributeNS(null, 'style', this.drawAnnotationStyle)
                this.thread.drawAnnotationSvg.appendChild(rect)
                return
            }
            
            // Clears existing highlight
            CSS.highlights.delete(this.highlightId)
            
            // Creates new text highlight
            let range
            if (this.thread) {
                range = this.thread.range.toRange()
            } else {
                range = this.range.toRange()
            }
            if (this.visible) {
                const highlight = new Highlight(range)
                CSS.highlights.set(this.highlightId, highlight)
                this.updateHighlightStyle()
            }
        },
    }
}
</script>
