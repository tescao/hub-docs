<script lang="ts">
	import type {
		WidgetProps,
		TableData,
		HighlightCoordinates,
	} from "../../shared/types";

	import { onMount } from "svelte";
	import WidgetTableInput from "../../shared/WidgetTableInput/WidgetTableInput.svelte";
	import WidgetSubmitBtn from "../../shared/WidgetSubmitBtn/WidgetSubmitBtn.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import { mod, parseJSON } from "../../../../utils/ViewUtils";
	import {
		addInferenceParameters,
		convertDataToTable,
		convertTableToData,
		getDemoInputs,
		getResponse,
		getSearchParams,
		updateUrl,
	} from "../../shared/helpers";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let shouldUpdateUrl: WidgetProps["shouldUpdateUrl"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	const columns: string[] = Object.keys(
		model?.widgetData?.[0]?.structuredData ?? {}
	);

	let computeTime = "";
	let error: string = "";
	let isLoading = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output: (string | number)[] = [];
	let outputJson: string;
	let table: (string | number)[][] = [columns];

	let highlighted: HighlightCoordinates = {};
	let highlightErrorKey = "";
	let scrollTableToRight: () => Promise<void>;
	let tableWithOutput: (string | number)[][];
	$: {
		const strucuredData = convertTableToData(table);
		if (output?.length) {
			strucuredData.Prediction = output;
			const lastColIndex = Object.keys(strucuredData).length - 1;
			highlighted = highlightOutput(output, lastColIndex);
			scrollTableToRight();
		} else {
			delete strucuredData.Prediction;
			highlighted = {};
			if (highlightErrorKey) {
				highlighted[highlightErrorKey] =
					"bg-red-100 border-red-100 dark:bg-red-800 dark:border-red-800";
				highlightErrorKey = "";
			}
		}
		tableWithOutput = convertDataToTable(strucuredData);
	}

	const COLORS = ["blue", "green", "yellow", "purple", "red"] as const;

	onMount(() => {
		const [dataParam] = getSearchParams(["structuredData"]);
		if (dataParam) {
			table = convertDataToTable((parseJSON(dataParam) as TableData) ?? {});
			getOutput();
		} else {
			const [demoTable] = getDemoInputs(model, ["structuredData"]);
			table = convertDataToTable((demoTable as TableData) ?? {});
			if (table && callApiOnMount) {
				getOutput({ isOnLoadCall: true });
			}
		}
	});

	function onChangeTable(updatedTable: string[][]) {
		table = updatedTable;
		output = [];
	}

	async function getOutput({
		withModelLoading = false,
		isOnLoadCall = false,
	} = {}) {
		for (let [i, row] of table.entries()) {
			for (const [j, cell] of row.entries()) {
				if (!String(cell)) {
					error = `Missing value at row=${i} & column='${columns[j]}'`;
					highlightErrorKey = `${--i}-${j}`;
					output = null;
					outputJson = "";
					return;
				}
				// tabular-data backend expects value `NaN` for "null value"
				if (/(null|nan)/i.test(String(cell))) {
					table[i][j] = "NaN";
				}
			}
		}

		// strip prediction column
		let { Prediction, ...tableWithoutOutput } =
			convertTableToData(tableWithOutput);

		if (shouldUpdateUrl && !isOnLoadCall) {
			updateUrl({
				data: JSON.stringify(tableWithoutOutput),
			});
		}

		const requestBody = {
			inputs: {
				data: tableWithoutOutput,
			},
		};
		addInferenceParameters(requestBody, model);

		isLoading = true;

		const res = await getResponse(
			apiUrl,
			model.id,
			requestBody,
			apiToken,
			parseOutput,
			withModelLoading,
			includeCredentials,
			isOnLoadCall
		);

		isLoading = false;
		// Reset values
		computeTime = "";
		error = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		output = null;
		outputJson = "";

		if (res.status === "success") {
			computeTime = res.computeTime;
			output = res.output;
			outputJson = res.outputJson;
		} else if (res.status === "loading-model") {
			modelLoading = {
				isLoading: true,
				estimatedTime: res.estimatedTime,
			};
			getOutput({ withModelLoading: true });
		} else if (res.status === "error") {
			error = res.error;
		}
	}

	function isValidOutput(arg: any): arg is (string | number)[] {
		return (
			Array.isArray(arg) &&
			arg.every((x) => typeof x === "string" || typeof x === "number")
		);
	}

	function parseOutput(body: unknown): (string | number)[] {
		if (isValidOutput(body)) {
			return body;
		}
		throw new TypeError(
			"Invalid output: output must be of type Array<string | number>"
		);
	}

	function highlightOutput(
		output: (string | number)[],
		colIndex: number
	): HighlightCoordinates {
		const set: Set<string | number> = new Set(output);
		let classes: Record<string, number> = {};
		if (set.size < COLORS.length) {
			classes = [...set].reduce((acc, cls, i) => ({ ...acc, [cls]: i }), {});
		}
		return output.reduce((acc, row, rowIndex) => {
			const colorIndex = classes[row] ?? mod(rowIndex, COLORS.length);
			const color = COLORS[colorIndex];
			acc[
				`${rowIndex}-${colIndex}`
			] = `bg-${color}-100 border-${color}-100 dark:bg-${color}-800 dark:border-${color}-800`;
			return acc;
		}, {});
	}

	function previewInputSample(sample: Record<string, any>) {
		table = convertDataToTable(sample.structuredData);
	}

	function applyInputSample(sample: Record<string, any>) {
		table = convertDataToTable(sample.structuredData);
		getOutput();
	}
</script>

<WidgetWrapper
	{apiUrl}
	{includeCredentials}
	{applyInputSample}
	{computeTime}
	{error}
	{isLoading}
	{model}
	{modelLoading}
	{noTitle}
	{outputJson}
	{previewInputSample}
>
	<svelte:fragment slot="top">
		<form>
			<div class="mt-4">
				{#if table.length > 1 || table[1]?.length > 1}
					<WidgetTableInput
						{highlighted}
						{isLoading}
						onChange={onChangeTable}
						table={tableWithOutput}
						canAddCol={false}
						bind:scrollTableToRight
					/>
				{/if}
			</div>
			<WidgetSubmitBtn
				{isLoading}
				onClick={() => {
					getOutput();
				}}
			/>
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom" />
</WidgetWrapper>
