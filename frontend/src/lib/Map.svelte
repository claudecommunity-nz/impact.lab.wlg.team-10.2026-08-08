<script lang="ts">
	import { onDestroy } from 'svelte';
	import type { Event } from '$lib/api';
	import { severityColor, worstSeverity } from '$lib/severity';

	interface Props {
		events: Event[];
		selectedId: string | null;
		onselect: (id: string) => void;
	}

	let { events, selectedId, onselect }: Props = $props();

	let container = $state<HTMLDivElement | undefined>();
	let map: import('leaflet').Map | undefined;
	let clusterGroup: import('leaflet').MarkerClusterGroup | undefined;
	let markers = new Map<string, import('leaflet').Marker>();

	const WELLINGTON_CENTER: [number, number] = [-41.2865, 174.7762];

	function severityIcon(
		L: typeof import('leaflet'),
		severity: string | null | undefined,
		selected: boolean
	) {
		const size = selected ? 22 : 16;
		const color = severityColor(severity);
		return L.divIcon({
			className: 'severity-marker',
			html: `<div style="width:${size}px;height:${size}px;border-radius:50%;background:${color};border:2px solid #1e293b;box-sizing:border-box;"></div>`,
			iconSize: [size, size]
		});
	}

	// Clusters are coloured by their worst-severity member, not an average —
	// a cluster of mostly-low reports with one high should still read as
	// urgent, not get diluted away.
	function clusterIcon(L: typeof import('leaflet'), cluster: import('leaflet').MarkerCluster) {
		const children = cluster.getAllChildMarkers();
		const severities = children.map((m) => (m.options as { severity?: string | null }).severity);
		const color = severityColor(worstSeverity(severities));
		const count = children.length;
		const size = 32 + Math.min(count, 20);
		return L.divIcon({
			className: 'severity-cluster',
			html: `<div style="width:${size}px;height:${size}px;border-radius:50%;background:${color};border:2px solid #1e293b;box-sizing:border-box;display:flex;align-items:center;justify-content:center;color:white;font-weight:600;font-size:12px;">${count}</div>`,
			iconSize: [size, size]
		});
	}

	function renderMarkers(L: typeof import('leaflet')) {
		if (!map) return;
		if (clusterGroup) {
			clusterGroup.clearLayers();
		} else {
			clusterGroup = L.markerClusterGroup({
				iconCreateFunction: (cluster) => clusterIcon(L, cluster)
			});
			map.addLayer(clusterGroup);
		}
		markers.clear();

		for (const event of events) {
			const { lat, lon } = event.location;
			if (lat == null || lon == null) continue;

			const isSelected = event.id === selectedId;
			const marker = L.marker([lat, lon], {
				icon: severityIcon(L, event.severity, isSelected)
			});
			(marker.options as { severity?: string | null }).severity = event.severity;
			marker.on('click', () => onselect(event.id));
			marker.bindTooltip(event.hazard_type ?? 'Report', { direction: 'top' });
			clusterGroup.addLayer(marker);
			markers.set(event.id, marker);
		}
	}

	$effect(() => {
		if (!container || map) return;
		let cancelled = false;

		// Leaflet touches `window`/`document` on init — dynamic import keeps
		// this out of the prerender/SSR path entirely (see dashboard/+page.ts).
		// leaflet.markercluster mutates the same `leaflet` module instance
		// (attaches markerClusterGroup onto it), so importing both then using
		// the first module's export is the correct pattern here.
		Promise.all([import('leaflet'), import('leaflet.markercluster')]).then(([L]) => {
			if (cancelled || !container) return;
			map = L.map(container).setView(WELLINGTON_CENTER, 12);
			L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
				attribution: '&copy; <a href="https://carto.com/attributions">CARTO</a>',
				maxZoom: 19
			}).addTo(map);
			renderMarkers(L);
		});

		return () => {
			cancelled = true;
		};
	});

	$effect(() => {
		// Re-render markers whenever events or the selection change, once the
		// map already exists (the effect above handles first creation).
		events;
		selectedId;
		if (map) {
			import('leaflet').then((L) => renderMarkers(L));
		}
	});

	onDestroy(() => {
		map?.remove();
	});
</script>

<div bind:this={container} class="h-full w-full"></div>
