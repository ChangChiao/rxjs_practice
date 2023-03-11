<script lang="ts">
    import { fromEvent } from 'rxjs';
    import {filter, map } from 'rxjs/operators';

    fromEvent(document, 'click')
    .pipe(
        filter((event: MouseEvent) => event.ctrlKey),
        map((e: MouseEvent) => {x: e.x; y: e.y;}),
    )
    .subscribe((data) => console.log('data', data))
    // .subscribe((position) => console.log(`x:${position.x} y:${position.y}`));
    
</script>