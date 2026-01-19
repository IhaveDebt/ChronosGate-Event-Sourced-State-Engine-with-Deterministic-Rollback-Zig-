const std = @import("std");

const Allocator = std.mem.Allocator;

const State = enum {
    Init,
    Active,
    Suspended,
    Closed,
};

const EventType = enum {
    Activate,
    Suspend,
    Resume,
    Close,
};

const Event = struct {
    id: usize,
    event_type: EventType,
};

const Engine = struct {
    allocator: Allocator,
    state: State,
    events: std.ArrayList(Event),

    pub fn init(allocator: Allocator) Engine {
        return Engine{
            .allocator = allocator,
            .state = State.Init,
            .events = std.ArrayList(Event).init(allocator),
        };
    }

    pub fn deinit(self: *Engine) void {
        self.events.deinit();
    }

    fn apply(self: *Engine, event: EventType) !void {
        switch (self.state) {
            .Init => if (event != .Activate) return error.IllegalTransition,
            .Active => if (event != .Suspend and event != .Close) return error.IllegalTransition,
            .Suspended => if (event != .Resume and event != .Close) return error.IllegalTransition,
            .Closed => return error.IllegalTransition,
        }

        self.state = switch (event) {
            .Activate => State.Active,
            .Suspend => State.Suspended,
            .Resume => State.Active,
            .Close => State.Closed,
        };

        try self.events.append(Event{
            .id = self.events.items.len + 1,
            .event_type = event,
        });
    }

    pub fn rollback(self: *Engine, to_event_id: usize) void {
        self.state = State.Init;

        var i: usize = 0;
        while (i < self.events.items.len and self.events.items[i].id <= to_event_id) : (i += 1) {
            _ = self.replay(self.events.items[i].event_type);
        }

        self.events.items.len = to_event_id;
    }

    fn replay(self: *Engine, event: EventType) bool {
        switch (event) {
            .Activate => self.state = State.Active,
            .Suspend => self.state = State.Suspended,
            .Resume => self.state = State.Active,
            .Close => self.state = State.Closed,
        }
        return true;
    }

    pub fn audit(self: *Engine) void {
        std.debug.print("\n=== Event Log ===\n", .{});
        for (self.events.items) |e| {
            std.debug.print("Event {d}: {any}\n", .{ e.id, e.event_type });
        }
        std.debug.print("Current State: {any}\n", .{self.state});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var engine = Engine.init(gpa.allocator());
    defer engine.deinit();

    try engine.apply(.Activate);
    try engine.apply(.Suspend);
    try engine.apply(.Resume);
    try engine.apply(.Close);

    engine.audit();

    std.debug.print("\nRolling back to event 2...\n", .{});
    engine.rollback(2);
    engine.audit();
}
