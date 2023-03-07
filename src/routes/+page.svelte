<script>
	import { writable } from 'svelte/store'
	import { createMachine, invoke, reduce, state, interpret, transition } from 'robot3'

	const svelteRobot = (machine) => {
		const { set: setState, ...state } = writable(machine.current)
		const { set: setContext, ...context } = writable(machine.context())

		const { send } = interpret(machine, (service) => {
			setState(service.machine.current)
			setContext(service.context)
		})

		return { context, send, state }
	}

	const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
	const loadUsers = async () => {
		await wait(1300)
		if (Math.random() > 0.7) throw new TypeError('loading users failed')
		return [
			{ id: 1, name: 'Wilbur' },
			{ id: 2, name: 'Xyler' },
			{ id: 3, name: 'Yanny' },
			{ id: 4, name: 'Zayn' }
		]
	}

	const {
		state: machineState,
		context: machineContext,
		send: machineTransition
	} = svelteRobot(
		createMachine({
			loading: invoke(
				loadUsers,
				transition(
					'done',
					'loaded',
					reduce((ctx, ev) => ({ ...ctx, users: ev.data }))
				),
				transition(
					'error',
					'error',
					reduce((ctx, ev) => ({ ...ctx, error: ev.error }))
				)
			),
			error: state(transition('retry', 'loading')),
			loaded: state()
		})
	)
</script>

{#if $machineState === 'loading'}
	<h1>Loading...</h1>
{:else if $machineState === 'loaded'}
	<h3>Users (online: {$machineContext.users.length})</h3>
	<ol>
		{#each $machineContext.users as user}
			<li>{user.name}</li>
		{/each}
	</ol>
{:else if $machineState === 'error'}
	<h2>
		There was a {$machineContext.error.name} (full description: {$machineContext.error.message})
		thrown while trying to load users
	</h2>
	<button on:click={() => machineTransition('retry')}>Retry</button>
{/if}