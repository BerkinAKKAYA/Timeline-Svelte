<script lang="ts">
	import { crossfade } from "svelte/transition";
	import { cubicInOut } from "svelte/easing";
	import { flip } from "svelte/animate";
	import DatePicker from "./DatePicker/DatePicker.svelte";

	const database = firebase.database();
	const auth = firebase.auth();
	const googleProvider = new firebase.auth.GoogleAuthProvider();

	let uid = 0;
	let data = [];
	let showDatePicker = false,
		showAddPopup = false;
	let selectedId = -1,
		selectedDate = new Date();
	let user = null;
	let promise = null;

	auth.onAuthStateChanged((u) => {
		user = u;

		if (user) {
			promise = database
				.ref("users/" + u.uid)
				.once("value")
				.then((ss) => {
					const fetched = ss.val() || {};

					if (Object.keys(fetched).length === 0) {
						const value = new Date(
							`01-01-${new Date().getFullYear() + 1}`
						).getTime();

						database.ref("users/" + u.uid).set({
							"End of the Year": value,
						});
						data = [{ id: uid++, key: "End of the Year", value }];
					} else {
						FetchData(fetched);
					}
				});
		}
	});

	const SignInGoogle = () => auth.signInWithPopup(googleProvider);
	const SignInAnonymously = () => auth.signInAnonymously();
	const SignOut = () => auth.signOut();

	function UpdateDatabase() {
		if (!user) {
			return console.log("Tried to update database without an user!");
		}

		const obj = {};
		for (const d of data) {
			obj[d["key"]] = d["value"];
		}

		database.ref("users/" + user.uid).set(obj);
	}

	function TimeLeft(date) {
		const diff = date - Date.now();
		const result = diff / (1000 * 60 * 60 * 24);
		return Math.abs(Math.ceil(result));
	}

	function ShowDatePicker(id) {
		const time = data.filter((x) => x["id"] === id)[0]["value"];
		selectedDate = new Date(time);
		selectedId = id;
		showDatePicker = true;
	}

	function UpdateDate() {
		for (const d of data) {
			if (d["id"] == selectedId) {
				d["value"] = selectedDate.getTime();
			}
		}

		data = data;
		selectedId = null;
		showDatePicker = false;
		SortData();
		UpdateDatabase();
	}

	function Remove(node) {
		const sure = confirm(`Are you sure to delete the item?`);

		if (sure) {
			data = data.filter((x) => x !== node);
		}

		UpdateDatabase();
	}

	function ShowAddPopup() {
		showAddPopup = true;
		setTimeout(InitializeDateSelector, 0);
	}

	function AddNew() {
		const name = document.querySelector("#add-popup input[type=text]");
		const date = document.querySelector("#add-popup input[type=date]");
		const nameVal = name.value;
		const dateVal = Date.parse(date.value);

		if (nameVal.length <= 0) {
			return alert("No Name Supplied...");
		}
		if (dateVal <= Date.now()) {
			return alert("Supplied Date is Already Over!");
		}

		// Is Already in List
		for (const x of data) {
			if (x["key"] === nameVal) {
				return alert("Event Already Exists.");
			}
		}

		data = [...data, { id: uid++, key: nameVal, value: dateVal }];
		showAddPopup = false;
		SortData();
		UpdateDatabase();
	}

	function InitializeDateSelector() {
		const d = new Date(1000 * 60 * 60 * 24 + +new Date());
		let month = "" + (d.getMonth() + 1),
			day = "" + d.getDate(),
			year = d.getFullYear();

		if (month.length < 2) month = "0" + month;
		if (day.length < 2) day = "0" + day;

		const addDateInput = document.querySelector(
			"#add-popup input[type=date]"
		);
		const today = `${year}-${month}-${day}`;
		addDateInput.value = today;
		addDateInput.min = today;
	}

	function SortData() {
		data.sort((a, b) => a.value - b.value);
	}

	function FetchData(fetchedData) {
		data = [];
		for (const [key, value] of Object.entries(fetchedData)) {
			data.push({ id: uid++, key, value });
		}
		SortData();
		data = data;
	}

	const [send, receive] = crossfade({
		fallback(node) {
			const tf = getComputedStyle(node).transform;
			return {
				easing: cubicInOut,
				css: (t) => `transform: ${tf} scale(${t})`,
			};
		},
	});

	document.addEventListener("keyup", (e) => {
		if (e.key === "Escape") {
			showAddPopup = false;
			showDatePicker = false;
		}
	});
	document.addEventListener("mousedown", (e) => {
		if (e.target.parentElement === document.body) {
			showAddPopup = false;
			showDatePicker = false;
		}
	});
	document.addEventListener("touchstart", (e) => {
		if (e.target.parentElement === document.body) {
			showAddPopup = false;
			showDatePicker = false;
		}
	});
</script>

{#if user}
	<DatePicker
		bind:visible={showDatePicker}
		bind:date={selectedDate}
		onConfirmDate={UpdateDate} />

	{#if showAddPopup}
		<div id="add-popup">
			<div id="container">
				<h1>ADD</h1>

				<input type="text" maxlength={20} placeholder="Event Name" />
				<input type="date" />
				<input type="submit" on:click={AddNew} />
			</div>
		</div>
	{/if}

	{#await promise}
		<p style="margin-top: 50px">Loading...</p>
	{:then}
		<div id="container">
			{#if data.length > 0}
				{#each data as days (days['id'])}
					<p
						in:receive={{ key: days['id'] }}
						out:send={{ key: days['id'] }}
						animate:flip>
						{#if days['value'] <= Date.now()}
							<div class="past" on:click={() => Remove(days)}>
								<span id="days">{TimeLeft(days['value'])}
									{TimeLeft(days['value']) > 1 ? 'days' : 'day'}</span>
								<span id="left">past since</span>
								<span id="keys">{days['key']}</span>
							</div>
						{:else}
							<span
								id="days"
								on:click={() => ShowDatePicker(days['id'])}>{TimeLeft(days['value'])}
								{TimeLeft(days['value']) > 1 ? 'days' : 'day'}
							</span>
							<span id="left">left for</span>
							<span
								id="keys"
								on:click={() => Remove(days)}>{days['key']}</span>
						{/if}
					</p>
				{/each}
			{:else}
				<p id="noData">Nothing to see here!</p>
			{/if}
		</div>
	{:catch}
		<p>ERROR!</p>
	{/await}

	<span id="add" on:click={ShowAddPopup}>ADD NEW</span>
	<span id="sign-out" on:click={SignOut}>Sign Out</span>
{:else}
	<div id="sign-in">
		<h1>Timeline</h1>
		<button on:click={SignInAnonymously}>Sign In Anonymously</button>
		<button on:click={SignInGoogle}>Sign In with Google</button>
	</div>
{/if}

<footer>
	<span>Created by</span>
	<a target="_blank" href="https://berkinakkaya.github.io">Berkin AKKAYA</a>
</footer>

<style lang="scss">
	#container {
		display: grid;
		margin-top: 50px;
		min-height: 70px;

		p {
			font-size: 3em;
			margin: 25px 0;

			@media (max-width: 1200px) {
				font-size: 4vw;
				margin: 2.5vw 0;
			}

			span {
				display: inline-block;
				transition: color 0.2s;
			}

			span#days {
				cursor: pointer;
				min-width: 4em;
				text-align: right;
				font-weight: 300;

				&:hover {
					color: green;
				}
			}

			span#left {
				font-weight: 100;
				color: #aaa;
				padding: 0 5px;
			}

			span#keys {
				cursor: pointer;
				font-weight: 400;

				&:hover {
					color: brown;
				}
			}

			.past {
				* {
					color: #aa0000 !important;
					transition: color 0.3s;
				}
				&:hover {
					cursor: pointer;

					* {
						color: #dd0000 !important;
					}
				}
			}
		}

		#noData {
			text-align: center;
			font-weight: 300;
			position: absolute;
			left: 50%;
			transform: translateX(-50%);
		}
	}
	#sign-out {
		color: #aa0000;
		margin-bottom: 50px;
		transition: color 0.3s ease;
		cursor: pointer;

		&:hover {
			color: #cc0000;

			&::after {
				width: 100%;
			}
		}
		&::after {
			content: "";
			width: 25%;
			height: 1px;
			display: block;
			background-color: #cc0000;
			margin-top: 5px;
			position: relative;
			left: 50%;
			transform: translateX(-50%);
			transition: width 0.3s ease;
		}
	}
	#add {
		margin: 50px 0;
		font-size: 2em;
		font-weight: 300;
		padding: 0 10px;
		transition: color 0.3s;
		opacity: 0.7;
		color: #333;
		cursor: pointer;

		&::after,
		&::before {
			content: "";
			display: block;
			width: 20%;
			height: 2px;
			margin: 5px;
			position: relative;
			background-color: green;
			transition: width 0.5s ease;
		}
		&::after {
			left: 100%;
			transform: translateX(-100%);
		}
		&:hover {
			color: darkgreen;

			&::after,
			&::before {
				width: calc(100%);
			}
		}

		@media (max-width: 400px) {
			font-size: 10vw;
		}
	}
	#add-popup {
		z-index: 1;
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0, 0, 0, 0.3);
		touch-action: pan-down;
		display: grid;
		place-items: center;
		backdrop-filter: blur(2px);
		transition: all 1s;

		#container {
			max-width: 600px;
			width: 80%;
			color: black;
			background: white;
			padding: 40px 0;
			display: flex;
			flex-direction: column;
			align-items: center;
			margin: 0;
			border-radius: 10px;
			box-shadow: 0 0 10px #666;

			h1 {
				text-align: center;
				font-weight: 400;
				margin-bottom: 15px;
			}

			input {
				max-width: 400px;
				width: 80%;
				font-size: 1.3em;
				padding: 15px;
				margin-top: 15px;
				border: 1px solid #999;
				outline: none;
				opacity: 0.6;
				color: black;
				transition: opacity 0.3s, border-radius 0.3s;

				&:hover {
					opacity: 1;
					border-radius: 5px;
				}

				&[type="submit"],
				&[type="date"],
				&[type="date"]::-webkit-calendar-picker-indicator {
					cursor: pointer;
				}
			}
		}
	}
	#sign-in {
		height: calc(100vh - 100px);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;

		h1 {
			text-align: center;
			font-weight: 400;
			margin-bottom: 50px;
		}

		button {
			width: 200px;
			padding: 15px 0;
			cursor: pointer;
		}
		button:not(:nth-child(2)) {
			margin-top: 10px;
		}
	}
	footer {
		margin-bottom: 50px;

		span {
			opacity: 0.6;
			font-weight: 300;
		}
		a {
			opacity: 0.6;
			font-weight: 400;
			color: black;
			transition: opacity 0.2s;

			&:hover {
				opacity: 1;
			}
		}
	}
</style>
