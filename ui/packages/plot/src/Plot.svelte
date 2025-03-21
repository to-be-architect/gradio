<script lang="ts">
	//@ts-nocheck
	import Plotly from "plotly.js-dist-min";
	import { Plot as PlotIcon } from "@gradio/icons";
	import { colors as color_palette, ordered_colors } from "@gradio/theme";
	import { get_next_color } from "@gradio/utils";
	import { Vega } from "svelte-vega";
	import { afterUpdate, beforeUpdate, onDestroy } from "svelte";
	import { create_config } from "./utils";
	import { Empty } from "@gradio/atoms";

	export let value;
	export let target;
	let spec = null;
	export let colors: Array<string> = [];
	export let theme: string;
	export let caption: string;
	export let bokeh_version: string  | null;

	function get_color(index: number) {
		let current_color = colors[index % colors.length];

		if (current_color && current_color in color_palette) {
			return color_palette[current_color as keyof typeof color_palette]
				?.primary;
		} else if (!current_color) {
			return color_palette[get_next_color(index) as keyof typeof color_palette]
				.primary;
		} else {
			return current_color;
		}
	}

	$: darkmode = theme == "dark";

	$: plot = value?.plot;
	$: type = value?.type;

	// Need to keep track of this because
	// otherwise embed_bokeh may try to embed before
	// bokeh is loaded
	$: bokeh_loaded = window.Bokeh === undefined

	function embed_bokeh(plot, type, bokeh_loaded){
		if (document){
			if (document.getElementById("bokehDiv")) {
				document.getElementById("bokehDiv").innerHTML = "";
			}
		}
		if (type == "bokeh" && window.Bokeh) {
			if (!bokeh_loaded) {
				load_bokeh();
				bokeh_loaded = true;
			}
			let plotObj = JSON.parse(plot);
			window.Bokeh.embed.embed_item(plotObj, "bokehDiv");
		}
	}

	$: embed_bokeh(plot, type, bokeh_loaded);

	$: if (type == "altair") {
		spec = JSON.parse(plot);
		const config = create_config(darkmode);
		spec.config = config;
		switch (value.chart || "") {
			case "scatter":
				if (spec.encoding.color && spec.encoding.color.type == "nominal") {
					spec.encoding.color.scale.range = spec.encoding.color.scale.range.map(
						(e, i) => get_color(i)
					);
				} else if (
					spec.encoding.color &&
					spec.encoding.color.type == "quantitative"
				) {
					spec.encoding.color.scale.range = ["#eff6ff", "#1e3a8a"];
					spec.encoding.color.scale.range.interpolate = "hsl";
				}
				break;
			case "line":
				spec.layer.forEach((d) => {
					if (d.encoding.color) {
						d.encoding.color.scale.range = d.encoding.color.scale.range.map(
							(e, i) => get_color(i)
						);
					}
				});
				break;
			case "bar":
				if (spec.encoding.color) {
					spec.encoding.color.scale.range = spec.encoding.color.scale.range.map(
							(e, i) => get_color(i)
						);
				}
				break;
			default:
				break;
		}
	}

	// Plotly
	let plotDiv;
	let plotlyGlobalStyle;

	const main_src = `https://cdn.bokeh.org/bokeh/release/bokeh-${bokeh_version}.min.js`;

	const plugins_src = [
		`https://cdn.pydata.org/bokeh/release/bokeh-widgets-${bokeh_version}.min.js`,
		`https://cdn.pydata.org/bokeh/release/bokeh-tables-${bokeh_version}.min.js`,
		`https://cdn.pydata.org/bokeh/release/bokeh-gl-${bokeh_version}.min.js`,
		`https://cdn.pydata.org/bokeh/release/bokeh-api-${bokeh_version}.min.js`
	];

	function load_plugins() {
		return plugins_src.map((src, i) => {
			const script = document.createElement("script");
			script.onload = () => initializeBokeh(i + 1);
			script.src = src;
			document.head.appendChild(script);

			return script;
		});
	}

	function load_bokeh() {
		const script = document.createElement("script");
		script.onload = handleBokehLoaded;
		script.src = main_src;
		document.head.appendChild(script);
		bokeh_loaded = true;
		return script;
	}

	function load_plotly_css() {
		if (!plotlyGlobalStyle) {
			plotlyGlobalStyle = document.getElementById("plotly.js-style-global");
			const plotlyStyleClone = plotlyGlobalStyle.cloneNode();
			target.appendChild(plotlyStyleClone);
			for (const rule of plotlyGlobalStyle.sheet.cssRules) {
				plotlyStyleClone.sheet.insertRule(rule.cssText);
			}
		}
	}

	const main_script = bokeh_version ? load_bokeh() : null

	let plugin_scripts = [];

	const resolves = [];
	const bokehPromises = Array(5)
		.fill(0)
		.map((_, i) => createPromise(i));

	const initializeBokeh = (index) => {
		if (type == "bokeh") {
			resolves[index]();
		}
	};

	function createPromise(index) {
		return new Promise((resolve, reject) => {
			resolves[index] = resolve;
		});
	}

	function handleBokehLoaded() {
		initializeBokeh(0);
		plugin_scripts = load_plugins();
	}

	afterUpdate(() => {
		if (type == "plotly") {
			load_plotly_css();
			let plotObj = JSON.parse(plot);
			plotObj.layout.title
				? (plotObj.layout.margin = { autoexpand: true })
				: (plotObj.layout.margin = { l: 0, r: 0, b: 0, t: 0 });
			Plotly.react(plotDiv, plotObj);
		}
	});

	onDestroy(() => {
		if (main_script in document.children) {
			document.removeChild(main_script);
			plugin_scripts.forEach((child) => document.removeChild(child));
		}
	});
</script>

{#if value && type == "plotly"}
	<div bind:this={plotDiv} />
{:else if type == "bokeh"}
	<div id="bokehDiv" class="gradio-bokeh"/>
{:else if type == "altair"}
	<div class="altair layout">
		<Vega {spec} />
		{#if caption}
			<div class="caption layout">
				{caption}
			</div>
		{/if}
	</div>
{:else if type == "matplotlib"}
	<div class="matplotlib layout">
		<!-- svelte-ignore a11y-missing-attribute -->
		<img src={plot} />
	</div>
{:else}
	<Empty size="large" unpadded_box={true}><PlotIcon /></Empty>
{/if}

<style>

	.gradio-bokeh {
		display: flex;
		justify-content: center;
	}

	.layout {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		width: var(--size-full);
		height: var(--size-full);
		color: var(--body-text-color);
	}
	.altair {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		width: var(--size-full);
		height: var(--size-full);
	}

	.caption {
		font-size: var(--text-sm);
	}

	.matplotlib img {
		object-fit: contain;
	}
</style>
