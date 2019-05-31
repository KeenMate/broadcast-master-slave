<template>
	<div id="app">
		<img alt="Vue logo" src="./assets/logo.png">
		<button @click="send">Send</button>
		{{title}}
	</div>
</template>

<script>
	// todo: handle on-close
	// todo: make master-slave architecture
	// todo: local storage communication

	import BroadcastMessage from './models/broadcastMessage'

	const VueWindowsStorageKey = 'vue-opened-windows'

	export default {
		name: 'app',
		data: () => ({
			isInitialized: false,

			isMaster: false,

			artifactId: (Math.random() * 100).toFixed(0),

			channel: new BroadcastChannel('MyChannel'),

			intervalId: -1,
			
			sharedState: {
				sendData: 0,
				counter: 0
			}
		}),
		computed: {
			title () {
				document.title = `${this.isMaster ? 'Master' : 'Slave'} - c: ${this.sharedState.counter}, s: ${this.sharedState.sendData}`
				
				return document.title
			}
		},
		mounted() {
			this.channel.onmessage = this.handleBroadcast
			this.channel.onmessageerror = (err) => console.log(err)

			this.doEcho()

			setTimeout(() => {
				// no response in meaningful time
				if (!this.isInitialized) {
					console.log('Not initialized in specified time (I guess Im Master)')

					this.isInitialized = true
					this.isMaster = true

					this.syncStorage()

					this.loadResources()
				}
			}, 20)

			window.addEventListener('beforeunload', ev => {
				console.log(ev)

				this.onClose()
			})
		},
		methods: {
			loadResources() {
				console.log('Initializing resources')

				this.intervalId = setInterval(() => {
					this.sharedState.counter++

					this.channel.postMessage(new BroadcastMessage('interval fired', null))
				}, 2000)
			},
			addHeirToStorage(heirId) {
				const openedTabs = JSON.parse(localStorage.getItem(VueWindowsStorageKey)) || []

				const index = openedTabs.indexOf(heirId)
				if (index === -1)
					openedTabs.push(heirId)

				console.log('Adding Heir to storage')
				localStorage.setItem(VueWindowsStorageKey, JSON.stringify(openedTabs))
			},
			removeHeirFromStorage(heirId) {
				let openedTabs = JSON.parse(localStorage.getItem(VueWindowsStorageKey)) || []

				const index = openedTabs.indexOf(heirId)
				if (index === -1)
					return

				openedTabs.splice(index, 1)

				console.log('Removing Heir from storage')
				localStorage.setItem(VueWindowsStorageKey, JSON.stringify(openedTabs))
			},
			promoteNextHeir() {
				let openedTabs = JSON.parse(localStorage.getItem(VueWindowsStorageKey)) || []

				if (openedTabs.length === 0) {
					console.log('Cannot promote next Master. There is no other heir (dynasty has fallen)')

					return
				}

				openedTabs.splice(0, 1) // openedTabs.indexOf(this.artifactId)

				localStorage.setItem(
					VueWindowsStorageKey,
					JSON.stringify(openedTabs)
				)

				console.log('promoting next master')
				this.channel.postMessage(new BroadcastMessage('next master', openedTabs[0]))
			},
			syncStorage() {
				const openedTabs = JSON.parse(localStorage.getItem(VueWindowsStorageKey)) || []

				if (openedTabs.indexOf(this.artifactId) === -1)
					openedTabs.push(this.artifactId)

				localStorage.setItem(
					VueWindowsStorageKey,
					JSON.stringify(openedTabs)
				)
			},
			handleBroadcast({data}) {
				console.log('Received message')

				if (data.address === 'echo') {
					console.log('It is ECHO message')
					if (!this.isMaster && this.isInitialized) {
						console.log('Im not master & Im initialized - wont handle it')						
						return
					}

					console.log('Im master so I will send my working data & add Heir to storage')
					this.channel.postMessage(new BroadcastMessage('echoecho', this.sharedState))
					this.addHeirToStorage(data.data)
				} else if (data.address === 'echoecho') {
					if (this.isInitialized)
						return
					
					console.log('It is echo echo message and now Im not initialized - Im not master now')

					this.isMaster = false
					this.sharedState = data.data
					
					this.isInitialized = true
				} else if (data.address === 'heir died') {
					if (!this.isMaster)
						return
					
					console.log('It is Heir died message & Im master so I will handle it')

					this.removeHeirFromStorage(data.data)
				} else if (data.address === 'next master') {
					if (data.data !== this.artifactId)
						return
					
					console.log('Its Next master & its me.. Im now new Master')

					this.isMaster = true

					// setup resources (stream hooks etc.)
					this.loadResources()
				} else if (data.address === 'send') {
					console.log('Its send message.. updating my data and setting documents title')

					this.sharedState.sendData = data.data.value

					document.title = data.data.value
				} else if (data.address === 'interval fired') {
					console.log('Its interval fired message.. syncing appropriate prop')

					this.sharedState.counter++
				} else {
					console.warn('channel error: message\'s address was not recognized (', data.address, ')')
				}
			},
			doEcho() {
				console.log('Doing ECHO')

				this.channel.postMessage(new BroadcastMessage('echo', this.artifactId))
			},
			onClose() {
				console.log('Im closing')

				if (this.isMaster) {
					console.log('And Im master - cleaning resources and promoting next heir')
					clearInterval(this.intervalId)

					this.promoteNextHeir()
				} else {
					console.log('I was just Heir - I will tell others about my death')
					this.channel.postMessage(new BroadcastMessage('heir died', this.artifactId))
				}
			},
			send() {
				console.log('Send data triggered')
				this.sharedState.sendData++

				document.title = this.sharedState.sendData

				this.channel.postMessage(new BroadcastMessage('send', {
					id: this.artifactId,
					value: this.sharedState.sendData
				}))
			}
		}
	}
</script>

<style>
	#app {
		font-family: 'Avenir', Helvetica, Arial, sans-serif;
		-webkit-font-smoothing: antialiased;
		-moz-osx-font-smoothing: grayscale;
		text-align: center;
		color: #2c3e50;
		margin-top: 60px;
	}
</style>
