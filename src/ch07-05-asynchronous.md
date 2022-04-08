# Asynchronous Idioms or "Push Notifications"

Push notifications are used when we want to be alerted of a truly unpredictable, asynchronous event that can happen at any time.

One of the main challenges of push notifications is not disclosing your `SID` to the notifying server. Remember, anyone with your `SID` can invoke any method on your server, including more sensitive ones.

The idiom here is to create and reveal a "single-purpose" server, whose sole job is to receive the push notification from the notifier, and forward this message back to the main server. The single purpose server exists on the `lib` side, and is thus the caller controls it and its construction. It runs in its own dedicated thread; thus, the single-purpose server spends most of its life blocked and not consuming CPU resources, and only springs to action once a notification arrives.

This pattern has the following properties:
- No disclosure of the main loop `SID`
- An extra "bounce" required for asynchronous notifications

The example below is taken from the `NetManager`'s wifi state change subscription service, and trimmed down to the core bits.

```rust,noplayground,ignore
// inside api.rs
// used for managing susbscriptions
#[derive(Debug, Archive, Serialize, Deserialize, Copy, Clone)]
pub(crate) struct WifiStateSubscription {
    // this is the "single-purpose" SID
    pub sid: [u32; 4],
    // this is the opcode dispatch number to use on the recipient side. Everyone
    // can have a different opcode table, so we must remember this with each SID.
    pub opcode: u32,
}

// all of the below sub-structures are `rkyv` serializeable
#[derive(Debug, Copy, Clone, rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]
pub struct WlanStatusIpc {
    pub ssid: Option<SsidRecord>,
    pub link_state: u16,
    pub ipv4: [u16; com_rs_ref::ComState::WLAN_GET_IPV4_CONF.r_words as usize],
}
// the `from_status()` method is a convenience trait that can take data from a native
// representation to an IPC-compatible version
impl WlanStatusIpc {
    pub fn from_status(status: WlanStatus) -> Self {
        WlanStatusIpc {
            ssid: status.ssid,
            link_state: status.link_state as u16,
            ipv4: status.ipv4.encode_u16(),
        }
    }
}

#[derive(num_derive::FromPrimitive, num_derive::ToPrimitive, Debug)]
pub(crate) enum Opcode {
    // add a subscriber to our push notifications
    SubscribeWifiStats,
    // remove a subscriber
    UnsubWifiStats,
    // this triggers a push notification; it's contrived for simplicity in this pared-down example
    StateChangeEvent,
    // ------ API opcodes to be discussed in detail below ------
    /// Exits the server
    Quit,
}

```
```rust,noplayground,ignore
// inside lib.rs
pub struct NetManager {
    netconn: NetConn,
    wifi_state_cid: Option<CID>,
    wifi_state_sid: Option<xous::SID>,
}
impl NetManager {
    pub fn new() -> NetManager {
        NetManager {
            netconn: NetConn::new(&xous_names::XousNames::new().unwrap()).expect("can't connect to Net Server"),
            wifi_state_cid: None,
            wifi_state_sid: None,
        }
    }
    pub fn wifi_state_subscribe(&mut self, return_cid: CID, opcode: u32) -> Result<(), xous::Error> {
        if self.wifi_state_cid.is_none() {
            let onetime_sid = xous::create_server().unwrap();
            let sub = WifiStateSubscription {
                sid: onetime_sid.to_array(),
                opcode
            };
            let buf = Buffer::into_buf(sub).or(Err(xous::Error::InternalError))?;
            buf.send(self.netconn.conn(), Opcode::SubscribeWifiStats.to_u32().unwrap()).or(Err(xous::Error::InternalError))?;

            // this thread is the "bouncer" that takes the status data and sends it on
            // to our local private server. Note that it only has two opcodes, which limits
            // the attack surface exposed to a ptoentially untrusted subscriber.
            self.wifi_state_cid = Some(xous::connect(onetime_sid).unwrap());
            self.wifi_state_sid = Some(onetime_sid);
            let _ = std::thread::spawn({
                let onetime_sid = onetime_sid.clone();
                let opcode = opcode.clone();
                move || {
                    loop {
                        let msg = xous::receive_message(onetime_sid).unwrap();
                        match FromPrimitive::from_usize(msg.body.id()) {
                            Some(WifiStateCallback::Update) => {
                                let buffer = unsafe {
                                    Buffer::from_memory_message(msg.body.memory_message().unwrap())
                                };
                                // have to transform it through the local memory space because you can't re-lend pages
                                let sub = buffer.to_original::<WlanStatusIpc, _>().unwrap();
                                let buf = Buffer::into_buf(sub).expect("couldn't convert to memory message");
                                buf.lend(return_cid, opcode).expect("couldn't forward state update");
                            }
                            Some(WifiStateCallback::Drop) => {
                                xous::return_scalar(msg.sender, 1).unwrap();
                                break;
                            }
                            _ => {
                                log::error!("got unknown opcode: {:?}", msg);
                            }
                        }
                    }
                    xous::destroy_server(onetime_sid).unwrap();
                }
            });
            Ok(())
        } else {
            // you can only hook this once per object
            Err(xous::Error::ServerExists)
        }
    }
    /// If we're not already subscribed, returns without error.
    pub fn wifi_state_unsubscribe(&mut self) -> Result<(), xous::Error> {
        if let Some(handler) = self.wifi_state_cid.take() {
            if let Some(sid) = self.wifi_state_sid.take() {
                let s = sid.to_array();
                send_message(self.netconn.conn(),
                    Message::new_blocking_scalar(Opcode::UnsubWifiStats.to_usize().unwrap(),
                    s[0] as usize,
                    s[1] as usize,
                    s[2] as usize,
                    s[3] as usize,
                    )
                ).expect("couldn't unsubscribe");
            }
            send_message(handler, Message::new_blocking_scalar(WifiStateCallback::Drop.to_usize().unwrap(), 0, 0, 0, 0)).ok();
            unsafe{xous::disconnect(handler).ok()};
        }
        Ok(())
    }
}
```

```rust,noplayground,ignore
// main-side code
#[xous::xous_main]
fn xmain() -> ! {
    // ... other stuff ...
    let mut wifi_stats_cache: WlanStatus = WlanStatus::from_ipc(WlanStatusIpc::default());
    let mut status_subscribers = HashMap::<xous::CID, WifiStateSubscription>::new();
    loop {
        let mut msg = xous::receive_message(sid).unwrap();
        // ... other opcodes ...
        Some(Opcode::SubscribeWifiStats) => {
            let buffer = unsafe {
                Buffer::from_memory_message(msg.body.memory_message().unwrap())
            };
            let sub = buffer.to_original::<WifiStateSubscription, _>().unwrap();
            let sub_cid = xous::connect(xous::SID::from_array(sub.sid)).expect("couldn't connect to wifi subscriber callback");
            status_subscribers.insert(sub_cid, sub);
        },
        Some(Opcode::UnsubWifiStats) => msg_blocking_scalar_unpack!(msg, s0, s1, s2, s3, {
            let sid = [s0 as u32, s1 as u32, s2 as u32, s3 as u32];
            let mut valid_sid: Option<xous::CID> = None;
            for (&cid, &sub) in status_subscribers.iter() {
                if sub.sid == sid {
                    valid_sid = Some(cid)
                }
            }
            xous::return_scalar(msg.sender, 1).expect("couldn't ack unsub");
            if let Some(cid) = valid_sid {
                status_subscribers.remove(&cid);
                unsafe{xous::disconnect(cid).ok();}
            }
        }),
        // contrived state change event. Use the below idiom whenever you need to send a push notification.
        Some(Opcode::StateChangeEvent) => {
            // ... other code to handle the state change

            // iterate through all the subscribers and send the notification
            for &sub in status_subscribers.keys() {
                let buf = Buffer::into_buf(WlanStatusIpc::from_status(wifi_stats_cache)).or(Err(xous::Error::InternalError)).unwrap();
                buf.send(sub, WifiStateCallback::Update.to_u32().unwrap()).or(Err(xous::Error::InternalError)).unwrap();
            }
        }
    }
}
```