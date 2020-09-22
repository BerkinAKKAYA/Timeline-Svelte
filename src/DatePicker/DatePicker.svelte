<script>
	/*
	FORKED FROM SharifClick/svelte-touch-datepicker
	*/
	import Switcher from "./Switcher.svelte";

	export let date = new Date();
	export let visible = false;
	export let years_map = [1900, 2100];
	export let classes = "";
	export let onConfirmDate = null;

	let years_count = years_map[1] - years_map[0] + 1;

	const MONTHS = [
		"Jan",
		"Feb",
		"Mar",
		"Apr",
		"May",
		"Jun",
		"July",
		"Aug",
		"Sept",
		"Oct",
		"Nov",
		"Dec",
	];
	const YEARS = new Array(years_count)
		.fill(years_map[0])
		.map((v, i) => v + i);
	const WEEKDAY = [
		"Sunday",
		"Monday",
		"Tuesday",
		"Wednesday",
		"Thursday",
		"Friday",
		"Saturday",
	];

	let _date;
	$: DAYS = new Array(
		new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate()
	)
		.fill(1)
		.map((v, i) => v + i);
	$: _date = date.toLocaleDateString("en-US");

	let resetDate = (event) => {
		event.stopPropagation();
		date = new Date();
	};

	let dateChanged = (event) => {
		let { type, changedData } = event.detail;
		let newDate = new Date();

		if (type === "day") {
			newDate = new Date(
				date.getFullYear(),
				date.getMonth(),
				changedData + 1
			);
		} else if (type === "month") {
			let maxDayInSelectedMonth = new Date(
				date.getFullYear(),
				changedData + 1,
				0
			).getDate();
			let day = Math.min(date.getDate(), maxDayInSelectedMonth);
			newDate = new Date(date.getFullYear(), changedData, day);
		} else if (type === "year") {
			let maxDayInSelectedMonth = new Date(
				years_map[1] + changedData,
				date.getMonth() + 1,
				0
			).getDate();
			let day = Math.min(date.getDate(), maxDayInSelectedMonth);
			newDate = new Date(1900 + changedData, date.getMonth(), day);
		}

		date = newDate;
	};

	function ToggleVisibility() {
		visible = !visible;
	}
</script>

<input
	type="text"
	class={classes}
	readonly
	value={_date}
	on:focus={ToggleVisibility}
	style="display: none" />
{#if visible}
	<div class="touch-date-popup">
		<div>
			<div class="touch-date-wrapper">
				<div class="date-line">
					{date.getDate()}
					{MONTHS[date.getMonth()]}
					{date.getFullYear()}
				</div>
				<p class="day-line">{WEEKDAY[date.getDay()]}</p>
				<div class="touch-date-picker">
					<Switcher
						type="day"
						data={DAYS}
						selected={date.getDate()}
						on:dateChange={dateChanged} />
					<Switcher
						type="month"
						data={MONTHS}
						selected={date.getMonth() + 1}
						on:dateChange={dateChanged} />
					<Switcher
						type="year"
						data={YEARS}
						selected={date.getYear() + 1}
						on:dateChange={dateChanged} />
				</div>
				<div class="touch-date-reset">
					<button on:click={ToggleVisibility}>Cancel</button>
					<button on:click={resetDate}>Today</button>
					<button
						on:click={() => {
							visible = !visible;

							if (onConfirmDate) {
								onConfirmDate(date);
							}
						}}>Ok</button>
				</div>
			</div>
		</div>
	</div>
{/if}

<style>
	.touch-date-popup {
		z-index: 1;
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.3);
		touch-action: pan-down;
	}
	.touch-date-popup > div {
		background: var(--svtd-popup-bg-color, white);
		color: var(--svtd-popup-color, black);
		margin-top: 30vh;
		width: 85%;
		max-width: 800px;
		border-radius: var(--svtd-popup-radius, 10px);
		left: 50%;
		transform: translateX(-50%);
		position: relative;
	}
	.touch-date-wrapper {
		display: flex;
		justify-content: center;
		align-items: center;
		flex-direction: column;
		font-size: var(--svtd-font-size, 20px);
		padding: 1.5rem;
	}

	.touch-date-picker {
		display: flex;
		padding: 50px 20px;
		margin: 10px 0;
		overflow: hidden;
	}

	.touch-date-reset {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 10px;
	}
	@media (max-width: 500px) {
		.touch-date-reset {
			grid-template-columns: 1fr;
		}
	}
	.touch-date-reset > button {
		width: 100px;
		height: 30px;
		border-radius: 15px;
		border: 1px solid grey;
		outline: none;
		color: grey;
		background: none;
		font-weight: 300;
		cursor: pointer;
		transition: color 0.3s, border-color 0.3s;
	}
	.touch-date-reset > button:hover {
		border-color: black;
		color: black;
	}
	.touch-date-reset button:nth-child(1):active {
		-webkit-transform: scale(0.95);
		transform: scale(0.95);
	}

	.date-line {
		font-size: 30px;
		font-weight: 300;
	}
	.day-line {
		margin: 10px 0;
	}

	@media (max-width: 600px) {
		.date-line {
			font-size: 1.1em;
		}
		.day-line {
			font-size: 0.9em;
		}
	}
</style>
