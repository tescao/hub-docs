<script lang="ts">
	import type { WidgetProps } from "../../shared/types";

	import { onMount } from "svelte";
	import WidgetAudioTrack from "../../shared/WidgetAudioTrack/WidgetAudioTrack.svelte";
	import WidgetFileInput from "../../shared/WidgetFileInput/WidgetFileInput.svelte";
	import WidgetOutputChart from "../../shared/WidgetOutputChart/WidgetOutputChart.svelte";
	import WidgetRecorder from "../../shared/WidgetRecorder/WidgetRecorder.svelte";
	import WidgetSubmitBtn from "../../shared/WidgetSubmitBtn/WidgetSubmitBtn.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import {
		getResponse,
		getBlobFromUrl,
		getDemoInputs,
	} from "../../shared/helpers";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	let computeTime = "";
	let error: string = "";
	let file: Blob | File | null = null;
	let filename: string = "";
	let fileUrl: string;
	let isLoading = false;
	let isRecording = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output: Array<{ label: string; score: number }> = [];
	let outputJson: string;
	let selectedSampleUrl = "";
	let warning: string = "";

	function onRecordStart() {
		file = null;
		filename = "";
		fileUrl = "";
		isRecording = true;
	}

	function onRecordError(err: string) {
		error = err;
	}

	function onSelectFile(updatedFile: Blob | File) {
		isRecording = false;
		selectedSampleUrl = "";

		if (updatedFile.size !== 0) {
			const date = new Date();
			const time = date.toLocaleTimeString("en-US");
			filename =
				"name" in updatedFile
					? updatedFile.name
					: `Audio recorded from browser [${time}]`;
			file = updatedFile;
			fileUrl = URL.createObjectURL(file);
		}
	}

	async function getOutput({
		withModelLoading = false,
		isOnLoadCall = false,
	} = {}) {
		if (!file && !selectedSampleUrl) {
			error = "You must select or record an audio file";
			output = [];
			outputJson = "";
			return;
		}

		if (!file && selectedSampleUrl) {
			file = await getBlobFromUrl(selectedSampleUrl);
		}

		const requestBody = { file };

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
		warning = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		output = [];
		outputJson = "";

		if (res.status === "success") {
			computeTime = res.computeTime;
			output = res.output;
			outputJson = res.outputJson;
			if (output.length === 0) {
				warning = "No classes were detected";
			}
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

	function isValidOutput(arg: any): arg is { label: string; score: number }[] {
		return (
			Array.isArray(arg) &&
			arg.every(
				(x) => typeof x.label === "string" && typeof x.score === "number"
			)
		);
	}

	function parseOutput(body: unknown): Array<{ label: string; score: number }> {
		if (isValidOutput(body)) {
			return body;
		}
		throw new TypeError(
			"Invalid output: output must be of type Array<label: string, score:number>"
		);
	}

	function applyInputSample(sample: Record<string, any>) {
		file = null;
		filename = sample.example_title;
		fileUrl = sample.src;
		selectedSampleUrl = sample.src;
		getOutput();
	}

	function previewInputSample(sample: Record<string, any>) {
		filename = sample.example_title;
		fileUrl = sample.src;
		output = [];
		outputJson = "";
	}

	onMount(() => {
		const [example_title, src] = getDemoInputs(model, ["example_title", "src"]);
		if (callApiOnMount && src) {
			filename = example_title ?? "";
			fileUrl = src;
			selectedSampleUrl = src;
			getOutput({ isOnLoadCall: true });
		}
	});
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
			<div class="flex items-center flex-wrap">
				<WidgetFileInput
					accept="audio/*"
					classNames="mt-1.5 mr-2"
					{onSelectFile}
				/>
				<span class="mt-1.5 mr-2">or</span>
				<WidgetRecorder
					classNames="mt-1.5"
					{onRecordStart}
					onRecordStop={onSelectFile}
					onError={onRecordError}
				/>
			</div>
			{#if fileUrl}
				<WidgetAudioTrack classNames="mt-3" label={filename} src={fileUrl} />
			{/if}
			<WidgetSubmitBtn
				classNames="mt-2"
				isDisabled={isRecording}
				{isLoading}
				onClick={() => {
					getOutput();
				}}
			/>
			{#if warning}
				<div class="alert alert-warning mt-2">{warning}</div>
			{/if}
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		<WidgetOutputChart classNames="pt-4" {output} />
	</svelte:fragment>
</WidgetWrapper>
