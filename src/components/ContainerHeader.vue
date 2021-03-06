<template>
    <div
        class="kiwi-header"
        v-bind:class="{
            'kiwi-header--showall': buffer_settings_open,
        }"
        @click="onHeaderClick"
    >

        <template v-if="isChannel()">
            <div class="kiwi-header-name">{{buffer.name}}</div>
            <div v-if="isJoined" class="kiwi-header-topic"></div>
            <div v-if="!isJoined" class="kiwi-header-notjoined">
                <a @click="joinCurrentBuffer" class="u-link">{{$t('container_join')}}</a>
            </div>
        </template>
        <template v-else-if="isServer()">
            <div class="kiwi-header-options">
                <a
                    class="u-button u-button-secondary"
                    @click="showNetworkSettings(buffer.getNetwork())"
                ><i class="fa fa-ellipsis-v" aria-hidden="true"></i></a>
            </div>
            <div class="kiwi-header-name">{{buffer.getNetwork().name}}</div>

            <div v-if="buffer.getNetwork().state === 'disconnected'" class="kiwi-header-server-connection">
                {{$t('container_notconnected')}}
                <a @click="buffer.getNetwork().ircClient.connect()" class="u-link">{{$t('connect')}}</a>
            </div>
            <div v-else-if="buffer.getNetwork().state === 'connecting'" class="kiwi-header-server-connection">
                {{$t('connecting')}}
            </div>
        </template>
        <template v-else-if="isQuery()">
            <div class="kiwi-header-options">
                <a class="u-button u-button-secondary" @click="closeCurrentBuffer">{{$t('close')}}</a>
            </div>
            <div class="kiwi-header-name">{{$t('container_privmsg', {user: buffer.name})}}</div>
        </template>
        <template v-else-if="isSpecial()">
            <div class="kiwi-header-options">
                <a class="u-button u-button-secondary" @click="closeCurrentBuffer">{{$t('close')}}</a>
            </div>
            <div class="kiwi-header-name">{{buffer.name}}</div>
        </template>

        <div
            v-if="buffer_settings_open"
            class="kiwi-header-buffersettings"
            @click.stop=""
        >

            <tabbed-view>
                <tabbed-tab :header="$t('settings')" :focus="true">
                    <channel-info v-bind:buffer="buffer"></channel-info>
                </tabbed-tab>
                <tabbed-tab :header="$t('banned')">
                    <channel-banlist v-bind:buffer="buffer"></channel-banlist>
                </tabbed-tab>
                <tabbed-tab :header="$t('notifications')">
                    <buffer-settings v-bind:buffer="buffer"></buffer-settings>
                </tabbed-tab>
            </tabbed-view>

            <a @click="buffer_settings_open=false" class="u-button u-button-secondary kiwi-header-close-buffersettings">
                <i class="fa fa-caret-up" aria-hidden="true"></i>
            </a>
        </div>
    </div>
</template>

<script>

import _ from 'lodash';
import state from 'src/libs/state';
import NetworkSettings from './NetworkSettings';
import BufferSettings from './BufferSettings';
import ChannelInfo from './ChannelInfo';
import ChannelBanlist from './ChannelBanlist';
import * as TextFormatting from 'src/helpers/TextFormatting';

export default {
    data: function data() {
        return {
            buffer_settings_open: false,
        };
    },
    props: ['buffer'],
    computed: {
        isJoined: function isJoined() {
            let buffer = this.buffer;
            return buffer.getNetwork().state === 'connected' && buffer.joined;
        },
    },
    components: {
        BufferSettings,
        ChannelInfo,
        ChannelBanlist,
    },
    methods: {
        formatMessage: function formatMessage(messageBody) {
            let words = messageBody.split(' ');
            words = words.map(word => {
                let parsed;

                parsed = TextFormatting.linkifyUrls(word, {
                    addHandle: true,
                    handleClass: 'fa fa-chevron-right kiwi-messagelist-message-linkhandle',
                });
                if (parsed !== word) return parsed;

                parsed = TextFormatting.linkifyChannels(word);
                if (parsed !== word) return parsed;

                return _.escape(word);
            });

            let parsed = words.join(' ');
            parsed = TextFormatting.ircCodesToHtml(parsed);

            return parsed;
        },
        isChannel: function isChannel() {
            return this.buffer.isChannel();
        },
        isServer: function isServer() {
            return this.buffer.isServer();
        },
        isQuery: function isQuery() {
            return this.buffer.isQuery();
        },
        isSpecial: function isSpecial() {
            return this.buffer.isSpecial();
        },
        showNetworkSettings: function showNetworkSettings(network) {
            state.$emit('active.component', NetworkSettings, {
                network,
            });
        },
        joinCurrentBuffer: function joinCurrentBuffer() {
            let network = this.buffer.getNetwork();
            this.buffer.enabled = true;
            network.ircClient.join(this.buffer.name);
        },
        closeCurrentBuffer: function closeCurrentBuffer() {
            state.removeBuffer(this.buffer);
        },
        onHeaderClick: function onHeaderClick(event) {
            let channelName = event.target.getAttribute('data-channel-name');
            if (channelName) {
                let network = this.buffer.getNetwork();
                state.addBuffer(this.buffer.networkid, channelName);
                network.ircClient.join(channelName);
            }
        },
    },
    watch: {
        buffer: function watchBuffer() {
            // When ever the buffer changes, close the settings dropdown
            this.buffer_settings_open = false;
        },
    },
};

</script>

<style>
.kiwi-header {
    box-sizing: border-box;
    z-index: 2;
    overflow: hidden;
}
.kiwi-header--showall {
    height: auto;
    max-height: 100%;
    overflow-y: auto;
}
.kiwi-header-name {
    display: inline-block;
}
.kiwi-header-notjoined {
    display: inline-block;
    margin-left: 1em;
}
.kiwi-header-server-settings {
    display: inline;
}
.kiwi-header-server-connection {
    display: inline-block;
}
.kiwi-header-options {
    display: inline-block;
    float: right;
    margin-top: 4px;
    margin-right: 4px;
}
.kiwi-header-close-buffersettings {
    float: right;
}
</style>
