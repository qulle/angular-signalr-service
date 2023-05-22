# Angular SignalR Service

## About

Service to connect to SignalR hub in ASP.NET and handling of internal message distribution. This example was written for a backend running ASP.NET framework meaning the client was forced to use jQuery and thus also the older SignalR lib that runs on jQuery.

## HTML

The main `index.html` needs to load both jQuery and jQuery-signalR minified scripts in the head section.
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8" />
        <title>SignalR Example</title>
        <base href="/" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <script src="./assets/lib/signalr/jquery-1.10.2.min.js"></script>
        <script src="./assets/lib/signalr/jquery.signalR-2.2.1.min.js"></script>
    </head>
    <body>
        <app-root></app-root>
    </body>
</html>
```

## Environments

You can make a dynamic service that loads some or all of these configurations at runtime, thus enabling the posibility to do changes without the need to make a new build of the application.
```typescript
export const environment = {
    signalR: {
        logging: true,
        useDefaultPath: true,
        reconnectTimeout: 1000,
        transport: 'auto',
        feed: {
            url: 'http://application-server/feed/eventhub/eh',
            name: 'EventHub',
        },
    },
};
```

## Component

The service can be started by initiating and starting the hub from the AppComponent.
```typescript
export class AppComponent implements OnInit {
    constructor(
        private readonly signalR: SignalRService
    ) {}
    
    onInit(): void {
        this.signalR.init().startConnection();
    }
}
```

## Service

The service will connect to the hub that is listed in the environment files.
```typescript
@Injectable({
    providedIn: 'root',
})
export class SignalRService {
    private readonly hubUrl: string;
    private readonly hubName: string;
    private readonly transport: string;
    private readonly logging: boolean;
    private readonly useDefaultPath: boolean;
    private readonly reconnectTimeout: number;

    // Internal SignalR state values
    private static readonly ConnectionState: Map<ConnectionState, string> = new Map([
        [ConnectionState.Connecting, 'connecting'],
        [ConnectionState.Connected, 'connected'],
        [ConnectionState.Reconnecting, 'reconnecting'],
        [ConnectionState.Disconnected, 'disconnected'],
    ]);

    private $: any;
    private connection: any;
    private hubProxy: any;

    constructor() {
        this.hubUrl = environment.feed.signalR.url;
        this.hubName = environment.feed.signalR.name;
        this.transport = environment.signalR.transport;
        this.logging = environment.signalR.logging;
        this.useDefaultPath = environment.signalR.useDefaultPath;
        this.reconnectTimeout = environment.signalR.reconnectTimeout; 

        // Get jQuery reference from Window object
        this.$ = (<any>window).jQuery;
    }

    private jQueryInstanceExists(): boolean {
        return this.$ !== undefined && this.$ !== null;
    }

    init(): SignalRService {
        if (!this.jQueryInstanceExists()) {
            throw new Error('No jQuery instance found');
        }

        this.connection = this.$.hubConnection(this.hubUrl, {
            useDefaultPath: this.useDefaultPath,
            logging: this.logging,
        });

        // SignalR Connection Lifetime Events
        this.connection.starting(this.onStarting.bind(this));
        this.connection.received(this.onReceived.bind(this));
        this.connection.connectionSlow(this.onConnectionSlow.bind(this));
        this.connection.reconnecting(this.onReconnecting.bind(this));
        this.connection.reconnected(this.onReconnected.bind(this));
        this.connection.stateChanged(this.onStateChanged.bind(this));
        this.connection.disconnected(this.onDisconnected.bind(this));
        this.connection.error(this.onError.bind(this));

        // Hub Setup (Not using the default created proxy)
        this.hubProxy = this.connection.createHubProxy(this.hubName);

        // Subscribe to all hub methods to get data from
        this.hubProxy.on('chat', this.onChat.bind(this));
        this.hubProxy.on('notifications', this.onNotification.bind(this));

        return this;
    }

    startConnection(): void {
        this.connection
            .start({
                transport: this.transport,
            })
            .done(this.onConnectionDone.bind(this))
            .fail(this.onConnectionFail.bind(this));
    }

    isConnected(): boolean {
        return this.connection && this.connection.state === ConnectionState.Connected;
    }

    getConnectionId(): string {
        return this.connection?.id || '-1';
    }

    getTransport(): string {
        return this.connection?.transport?.name || 'None';
    }

    //-------------------------------------------------------------
    // Section: Connection Lifetime Events
    //-------------------------------------------------------------

    private onConnectionDone(): void {
        console.dir({
            method: 'onConnectionDone',
            id: this.connection.id,
            tansport: this.connection.transport.name,
        });
    }

    private onConnectionFail(error: any): void {
        console.dir({
            method: 'onConnectionFail',
            error
        });
    }

    private onStarting(): void {
        console.dir({
            method: 'onStarting'
        });
    }

    private onReceived(data: any): void {
        console.dir({
            method: 'onReceived',
            data
        });
    }

    private onConnectionSlow(): void {
        console.dir({
            method: 'onConnectionSlow'
        });
    }

    private onReconnecting(): void {
        console.dir({
            method: 'onReconnecting'
        });
    }

    private onReconnected(): void {
        console.dir({
            method: 'onReconnected',
            id: this.connection.id,
            tansport: this.connection.transport.name,
        });
    }

    private onStateChanged(data: any): void {
        const oldState = SignalRService.ConnectionState.get(data.oldState) || 'N/A';
        const newState = SignalRService.ConnectionState.get(data.newState) || 'N/A';

        console.dir({
            method: 'onStateChanged',
            state: {
                old: `${data.oldState}:${oldState}`,
                new: `${data.newState}:${newState}`
            }
        });
    }

    private onDisconnected(): void {
        console.dir({
            method: 'onDisconnected',
            reason: this.connection.lastError?.message || 'Unkown'
        });

        // The connection was dropped, make a new connection
        window.setTimeout(() => {
            this.startConnection();
        }, this.reconnectTimeout);
    }

    private onError(errorData: any, sentData: any): void {
        console.dir({
            method: 'onError',
            data: {
                error: errorData,
                sent: sentData
            }
        });
    }

    //-------------------------------------------------------------
    // Section: Server Invoked Methods
    //-------------------------------------------------------------

    private onChat(data: ChatMessage): void {
        console.dir({
            method: 'onChat',
            data: data
        });
    }

    private onNotification(notification: NotificationMessage): void {
        console.dir({
            method: 'onNotification',
            notification: notification
        });
    }
}
```

The service above uses the following types and data models.
```typescript
export enum ConnectionState {
    Connecting = 0,
    Connected = 1,
    Reconnecting = 2,
    Disconnected = 4,
}

export interface ChatMessage {
    data: string;
    timestamp: string;
    hash: string;
}

export interface NotificationMessage {
    level: string;
    data: string;
    timestamp: string;
    hash: string;
}
```

A good way of distributing messages internally in the application is to use a Subject as a messagestream. That way every artifact in the application can send messages that other artifacts can subscribe to.

## Author
[Qulle](https://github.com/qulle/)
