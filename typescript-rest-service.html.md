

# How to Build a Rest Network Service in TypeScript for Angular2

### Goals

If you ever decided to take seriosly the [REST guidelines](https://books.google.hu/books?id=XUaErakHsoAC&printsec=frontcover&source=gbs_ge_summary_r&cad=0#v=onepage&q&f=false), you can easily find yourself managing different objects in a similar way: most of your objects can be listed, deleted, updated, retrieved by ID, or maybe a new instance created ([CRUD operations](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)) or maybe patched (partially updated). 

To avoid code duplication (keep your code [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)), let us identify the common used cases which can be managed in our general class:

* CRUD operations
* lightweight cache: build a cache, update it properly and provide service to cleanup. Additionally it's nice to have some sorting options to keep the cached list consequently sorted (I like to have the model itself resonable sorted, sorting is an expensive operations, in worst case we need to resort it in the view component (as it [is recommended by Angular Team](https://angular.io/docs/ts/latest/guide/pipes.html#!#no-filter-pipe)), oterwise it's ready to use/display).
* URL for the requests

Because our project is an Angular2 - TypeScript project, and Angular2 comes with RxJS, let's add some specificum of this tech-stack:

* provide a stream the subscribers to be notified about the cache changes
* manage network requests as Promises
* keep type-safety for managing different objects

## Some notes about the REST URLs

The URLs of a REST service can be specified using the following pattern:
path_of_items/<itemID>/<path_of_subItems>/<subItemID>/...

An example to make it more clearer:
Let's say I want to list all of the Shops in a given City. In this case the base URL can be e.g:
```javascript
/api/cities/12/shops
```

In this case the CRUD service will look like:

* Create                    - POST shop details (JSON) to /api/cities/12/shops
* Read (all shops)          - GET /api/cities/12/shops
* Update a shop             - PUT shop details (JSON) to /api/cities/12/shops/<shopID>
* Delete                    - DELETE /api/cities/12/shops
* Read (one shop's deails)  - GET /api/cities/12/shops/<shopID>

You can see the base URL (`/api/cities/12/shops`) they are static, however it has a moving part which is the nested IDs (cityID in our example). So even if our service can produce the proper URL for the given CRUD service, it will not be totally stateless: it will have to take some IDs to build the URL correctly. Even if it could be managed, I decided to let an Incejtable Service to manage the URL, as a result all my network request can be fully stateless.

### Implementation

Let's see how it is implemented:

```javascript

import...

/* Every REST instance must be able to identify itself! */
export interface Identifiable {
    id: any;
}

/* Sort Order specifying wether the results array needs to be descending or ascending sorted */
export enum SortOrder {
    ASC = 1,
    DESC
}

/* A SortDescriptor to specify the name of the attribute & the sort order to sort a list */
export interface SortDescriptor {
    attribute: string,
    order: SortOrder
}


/**
 * Generic REST network service to interact with the REST APIs. The type of object must be specified
 * - create, retrieve, update, patch, delete an item
 * - get a list of items
 * - keep the list of items properly sorted
 */
export class RestService<T extends Identifiable> extends BaseNetworking {

    /* Represent the list of the requested/updated/created Items, that changes over time.
     * Lightweight cache: Subscribers will receive the last (or initial) list when subscribing (and will get all the subsequent notifications)
     */
    private dataSubject$: BehaviorSubject<T[]> = null;

    /*
     * Describe the expected sorting behaviour of the requested data.
     **/
    private _sortDescriptors: SortDescriptor[] = [];

    /*
     * Specify how to sort the results.
     *
     * E.g
     * Format:
     * [
     *   {"attribute": "created", "order": "asc"},
     *   {"attribute": "name", "order": "desc"}
     * ]
     */
    set sortDescriptors(sortDescriptors: SortDescriptor[]) {
        this._sortDescriptors = sortDescriptors;

        //Cache attributes & sorting orders (as strings)
        this.sortAttributes = _.map<SortDescriptor, string>(sortDescriptors, 'attribute');
        this.sortOrders = _.map<SortDescriptor, SortOrder>(sortDescriptors, 'order').map(order => SortOrder[order].toLowerCase());
    }

    get sortDescriptors(): SortDescriptor[] {
        return this._sortDescriptors;
    }

    /* Cache attributes & sort orders (only for optimization) */
    private sortAttributes: string[] = [];
    private sortOrders: string[] = [];

    constructor() {
        super();
        this.dataSubject$ = new BehaviorSubject([]);
    }

    /**
     * Publish observables that every subscribers will be notified of
     */
    get data$() {
        return this.dataSubject$.asObservable();
    }

    /*
     * Request a list of object
     */
    list(url: string) {
        return this.baseGet(url).then(
            data => {
                this.updateStream(data);
                return data;
            }
        );
    }

    /*
     * Update an object (all of the required attributes must be presented in `savingInstance`)
     */
    update(updateURL: string, savingInstance: Identifiable): Promise<T> {
        return this.basePut(
            updateURL,
            savingInstance
        ).then(
            data => {
                this.replaceCachedItem(data);
                return data;
            }
        );
    }

    /*
     * Create an new object
     */
    delete(url: string, instance_id: any): Promise<void> {
        return this.baseDelete(url).then(
            () => {
                const contents = this.getLastData();
                _.remove(contents, (data) => data.id === instance_id );
                this.updateStream(contents);
            }
        );
    }

    /*
     * Delete an object
     */
    create(url: string, eventContent: Identifiable): Promise<T> {
        return this.basePost(
            url,
            eventContent
        ).then(
            data => {
                this.updateStream(_.concat(this.getLastData(), data));
                return data;
            }
        );
    }

    /*
     * Retrieve an object
     */
    retrieveItem(url: string): Promise<T> {

        const replaceOrPushItem = (item) => {
            // FIXME: This way doesn't seem to be very elegant and functional
            const contents = this.getLastData();
            const currentIndex = _.findIndex(contents, { id: item.id });
            if (currentIndex != -1) {
                contents[currentIndex] = item;
            }
            else {
                contents.push(item);
            }
            this.updateStream(contents);
        };

        return this.baseGet(url).then(
            data => {
                replaceOrPushItem(data);
                return data;
            }
        );
    }

    /*
     * Patch an object (update a few attributes only)
     */
    patch(url: string, item: any): Promise<T> {
        return this.basePatch(
            url, item
        ).then(
            data => {
                this.replaceCachedItem(data);
                return data;
            }
        );
    }

    /*
     * Clean the cache
     */
    clean() {
        this.dataSubject$.next([]);
    }

    /******
     * Internal cache management
     ******/

    private updateStream(data: T[]) {

        if (this.sortDescriptors.length) {
            data = _.orderBy(data, this.sortAttributes, this.sortOrders);
        }

        this.dataSubject$.next(data);
    }

    private replaceCachedItem(item) {
        const contents = this.getLastData();
        const index = _.findIndex(contents, { id: item.id });
        if (index  != -1) {
            contents[index] = item;
            this.updateStream(contents);
        }
    }

}

```

`Base Networking` is responsible for all required HTTP requests (GET, POST, PATCH, PUT, DELETE). Also, it takes care of the refresh and the auth token whenever the server requires to update it.

I used `BehaviorSubject` to create a lightweight cache: Subscribers will receive the last (or initial) list when subscribing (and will get and all the subsequent notifications). It is a good choice, because our initial data is straight forward (and empty list of items), otherwise we can still use [`ReplaySubject`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/subjects/replaysubject.md).

You have beter control on the available functionality, for instance, if you need just list the of objects, update, create, delete... services won't be available unlike inheritance.

I used it in a network service:


```javascript
export interface Shop {
    id: number;
    name: Date;
    populationSize?: number;
}
```

```javascript

import ...

@Injectable()
export class CityShopsService {

    private restService: RestService<Shop> = null;

    readonly cityShops$: Observable<Shop[]> = null;

    constructor() {
        // Craete Rest list service
        this.restService = new RestService<Shop>();
    
        // Order shops by it's population size
        this.restService.sortDescriptors = [ { attribute: "populationSize", order: SortOrder.ASC } ];

        // Hook up data & cache cleanup
        this.cityShops$ = this.restService.data$;
        this.clean = this.restService.clean.bind(this.restService);
    }

    private URL(cityID, shopID = null): string {
        const baseURL = `${environment.api_prefix}/e/cities/${cityId}/shops/`;
        return shopID ? baseURL + shopID : baseURL;
    }

    listShops(cityId: number) {
        this.restService.list(this.URL(cityId));
    }

    updateShop(cityId: number, savingInstance: Shop) {
        this.restService.update(this.URL(cityId, savingInstance.id), savingInstance);
    }

    deleteShop(cityId: number, shopID: number): Promise<void> {
        this.restService.delete(this.URL(cityId, shopID), shopID);
    }

    createShop(cityId: number, shop: Shop): Promise<Shop>  {
        this.restService.create(this.URL(cityId), shop);
    }

    getShop(cityId: number, shopID: number) {
        this.restService.retrieveItem(this.URL(cityId, shopID));
    }

    clean: () => void;

}
```

Maybe we could adjust the `RestService<T>` to be a proper "base-class" of the `CityShopsService`, like `export class CityShopsService extend RestService<Shop>`, and use the super's services directly. In this situation I would have no control on how to exclude some services, for example if I want to have only reading services but no update/patch/delete services, and also, the URL building will be much more complicated and hard to maintain because of the moving parts of the URL (e.g /cities/<cityID>/shops/<shopID>/products/<productID> ...). We can adjust the services to accept open-ended parameter list, but then we need to manage to separate URL building parameters from the updating/deleting object, which also can be confusing. Beside the object which would use the `CityShopsService`'s services won't know the proper signiture of these services, so we would have less descriptive services then now. Same reasons why `RestService` is far from ideal to be used as a [mixin](https://www.typescriptlang.org/docs/handbook/mixins.html)

So even if the subclasses can be smaller and quicker to implement(would be enough to override only one URL builder function), but the usability and flexibility of the class would pay its price... That is why I decided to use [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance).

Let's see a simple example on how to use `CityShopsService`. This is the standard way, nothing special here, I won't be including the edit/create features, I will just give you the insights on how the data-driven architecture works by using only the cache (and silently updating it. As you can see, no need to wait for promises, the cached `cityShops` will drive all the changes to the UI.

```javascript

import...


@Component({
    selector: 'city-shops-timeline',
    template: `
        <div *ngFor="let shop of eventShops | async ">
            {{shop.name}} - {{shop.populationSize}}
            <button type="button" (click)="onDeleteShop(shop)">Delete</button>
        </div>
    `,
    styleUrls: [
        'city-shops-timeline.component.css'
    ]
})
export class EventBinsTimelineComponent implements OnInit {
    @Input()
    city: City;

    cityShops: Observable<Shop[]>;

    constructor(private cityShopsService: CityShopsService) { }

    ngOnInit() {
        // Let's get the shops from cache
        this.eventBins = this.cityShopsService.cityShops$;

        // Silently update the cache. Data Driven: the `cityShops$` will be updated, no need to wait for Promises!
        this.cityShopsService.listShops(this.event.id);
    }

    onDeleteShop(shop: Shop) {
        // Data Driven: the cache will be updated, as a result `cityShops$` will immediately be changed, no need to wait for Promises.
        this.cityShopsService.deleteShop(this.city.id, shop.id);
    }

}

```

@Balazs Nemeth senior iOS & web-fronted developer
[LinkedIn](https://www.linkedin.com/in/runningios/)
