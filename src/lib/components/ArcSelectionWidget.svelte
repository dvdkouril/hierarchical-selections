<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import { arc, pie } from 'd3-shape';
	import { randomNiceColor } from '$lib/util';
	import type { Widget } from '$lib/widget';

	const dispatch = createEventDispatcher();

	export let width: number;
	export let height: number;
	export let N: number;
	// export let widgetId;
	export let colors: string[];
	// export let selectionColorsRange;
	export let colorForSelection = null; //~ if this is null, generate new color; otherwise use this one
	export let selections = [];
	export let widget: Widget;

	let selectionInProgress = false;
	export let hoveredBin = null;

	$: pieceSize = width / bins.length;
	$: bins = [...Array(N).fill(1)];

	$: radius = Math.min(width, height) / 2;
	export let widgetThickness = 25;

	const arcGen = arc();
	$: arcs = pie()(bins);

	$: segments = arcs.map((arc) => {
		let input = {
			innerRadius: radius - widgetThickness,
			outerRadius: radius,
			startAngle: arc.startAngle,
			endAngle: arc.endAngle
		};
		return arcGen(input);
	});

	$: selectionsArcs = selections.map((sel) =>
		arcGen({
			innerRadius: radius - widgetThickness,
			outerRadius: radius,
			startAngle: arcs[sel.start].startAngle,
			endAngle: arcs[sel.end].endAngle
		})
	);

	function pickSelectionColor() {}

	const mouseOvered = (event) => {
		hoveredBin = parseInt(event.target.id.split('-')[1]); //~ this is bit of a weird solution...maybe fix later
		//~ multiple selections version
		if (selectionInProgress) {
			const binId = parseInt(event.target.id.split('-')[1]); //~ this is bit of a weird solution...maybe fix later
			const activeSelection = selections.slice(-1)[0];
			const selectionsMinusLast = selections.slice(0, selections.length - 1);
			//~ figure out which direction the selection is
			if (binId < activeSelection.start) {
				selections = [...selectionsMinusLast, { ...activeSelection, start: binId }];
			} else {
				selections = [...selectionsMinusLast, { ...activeSelection, end: binId }];
			}
		}
	};

	const mouseOut = (event) => {
		hoveredBin = null;
	};

	const mouseDown = (event: MouseEvent) => {
		console.log('Selection started.');
		const binId = event.target.id.split('-')[1];
		// const selColor = randomNiceColor();
		// const selColor = "red";
		// const selColor = randomColorFromRange(selectionColorsRange);
		const selColor = colorForSelection == null ? randomNiceColor() : colorForSelection;
		selections.push({ start: parseInt(binId), end: parseInt(binId), color: selColor });
		selectionInProgress = true;
	};

	const mouseUp = (event) => {
		//~ => selection finished
		console.log('Selection ended.');
		selectionInProgress = false;
		dispatch('selectionFinished', {
			selection: selections.slice(-1)[0],
			// sourceWidget: widgetId,
			sourceWidget: widget
		});
	};

	const touchStart = (event: TouchEvent) => {
		console.log('touch start.');
		switch (event.touches.length) {
			case 1:
				if (event.target == undefined) {
					break;
				}
				if (event.target instanceof Element) {
					const binId = event.target.id.split('-')[1];
					const selColor = colorForSelection == null ? randomNiceColor() : colorForSelection;
					selections.push({ start: parseInt(binId), end: parseInt(binId), color: selColor });
					selectionInProgress = true;
				}
				break;
			// case 2: break;
			default:
				break;
		}
	};
	const touchEnd = (event: TouchEvent) => {
		//~ => selection finished
		console.log('touch end.');
		selectionInProgress = false;
		dispatch('selectionFinished', {
			selection: selections.slice(-1)[0],
			// sourceWidget: widgetId,
			sourceWidget: widget
		});
	};
	const touchMove = (event: TouchEvent) => {
		console.log('touch move');
		event.preventDefault();
		event.stopPropagation();
        const firstTouch = event.touches[0];
        const elUnderTouch = document.elementFromPoint(firstTouch.clientX, firstTouch.clientY);
        if (elUnderTouch == null) {
            return;
        }
		hoveredBin = parseInt(elUnderTouch.id.split('-')[1]); //~ this is bit of a weird solution...maybe fix later
		//~ multiple selections version
		if (selectionInProgress) {
			console.log('hoveredBin:' + hoveredBin);
			const binId = parseInt(elUnderTouch.id.split('-')[1]); //~ this is bit of a weird solution...maybe fix later
			const activeSelection = selections.slice(-1)[0];
			const selectionsMinusLast = selections.slice(0, selections.length - 1);
			//~ figure out which direction the selection is
			if (binId < activeSelection.start) {
				selections = [...selectionsMinusLast, { ...activeSelection, start: binId }];
			} else {
				selections = [...selectionsMinusLast, { ...activeSelection, end: binId }];
			}
		}
	};
</script>

<div id="arc-selection-widget" style="position: absolute; z-index: 2; pointer-events: none">
	<svg
		{width}
		{height}
		viewBox={`${-width / 2} ${-height / 2} ${width} ${height}`}
		pointer-events="none"
	>
		{#each segments as bin, i}
			<path
				d={bin}
				id={'bin-' + i}
				fill={i == hoveredBin ? 'red' : colors[i]}
				pointer-events="all"
				on:mousedown={mouseDown}
				on:mouseup={mouseUp}
				on:mouseover={mouseOvered}
				on:mouseout={mouseOut}
				on:touchstart={touchStart}
				on:touchend={touchEnd}
				on:touchmove={touchMove}
				on:focus={() => {}}
			/>
		{/each}
		<!-- Selection indication overlay -->
		{#if selections.length > 0}
			{#each selectionsArcs as selArc, i}
				<path
					d={selArc}
					id={'selection-arc-' + i}
					style="stroke-width: 5px; stroke: {selections[i].color}; fill: none; pointer-events:none"
				/>
			{/each}
		{/if}
	</svg>
</div>
