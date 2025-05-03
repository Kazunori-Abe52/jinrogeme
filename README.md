# jinrogeme



@SpringBootApplication
public class WerewolfGameApplication {
    public static void main(String[] args) {
        SpringApplication.run(WerewolfGameApplication.class, args);
    }
}


@RestController
public class LineWebhookController {

    private final GameEngine game = new GameEngine(LineMessagingClient.builder("YOUR_CHANNEL_TOKEN").build(), "GROUP_ID");

    @PostMapping("/webhook")
    public ResponseEntity<Void> callback(@RequestBody LineEvent event) {
        String text = event.getText();
        String userId = event.getUserId();
        String displayName = event.getDisplayName();

        switch (text) {
            case "/join" -> {
                game.addPlayer(userId, displayName);
                game.sendPrivateMessage(userId, "参加ありがとうございます！");
            }
            case "/start" -> {
                game.startGame();
            }
            case "/guard" -> {
                game.sendQuickReply(userId, "誰を護衛しますか？", game.getAlivePlayerDisplayNames(userId));
            }
            case "/inspect" -> {
                game.sendQuickReply(userId, "誰を占いますか？", game.getAlivePlayerDisplayNames(userId));
            }
        }

        return ResponseEntity.ok().build();
    }
}

public class GameEngine {

    enum Role { VILLAGER, WEREWOLF, SEER, MEDIUM, KNIGHT, MADMAN }
    enum Phase { SETUP, ASSIGN, NIGHT, DISCUSS, VOTE, EXECUTE, CHECK, END }

    static class Player {
        String id, name;
        boolean alive = true;
        Role role;
        String voteTarget, guardTarget, inspectTarget;
        Player(String id, String name) {
            this.id = id;
            this.name = name;
        }
    }

    private final Map<String, Player> players = new HashMap<>();
    private final LineMessagingClient client;
    private final String groupId;
    private Phase phase = Phase.SETUP;
    private String lastExecuted = null;

    public GameEngine(LineMessagingClient client, String groupId) {
        this.client = client;
        this.groupId = groupId;
    }

    public void addPlayer(String id, String name) {
        players.put(id, new Player(id, name));
    }

    public void startGame() {
        assignRoles();
        phase = Phase.NIGHT;
        sendGroup("\uD83C\uDF19 夜が来ました。役職の方は行動を選択してください。");
    }

    public void assignRoles() {
        List<Role> roleList = List.of(Role.WEREWOLF, Role.SEER, Role.MEDIUM, Role.KNIGHT, Role.MADMAN);
        List<Player> shuffled = new ArrayList<>(players.values());
        Collections.shuffle(shuffled);
        int i = 0;
        for (Player p : shuffled) {
            p.role = i < roleList.size() ? roleList.get(i) : Role.VILLAGER;
            i++;
            sendPrivate(p.id, "あなたの役職は " + p.role + " です。");
        }
    }

    public void processNight() {
        String guarded = players.values().stream()
            .filter(p -> p.role == Role.KNIGHT && p.guardTarget != null)
            .map(p -> p.guardTarget).findFirst().orElse(null);

        Player target = players.values().stream()
            .filter(p -> p.role != Role.WEREWOLF && p.alive).findFirst().orElse(null);

        if (target != null && !target.id.equals(guarded)) {
            target.alive = false;
            sendGroup("\uD83D\uDC80 今夜 " + target.name + " が襲撃されました。");
        } else {
            sendGroup("\uD83D\uDE0C 今夜は誰も襲われませんでした。");
        }

        for (Player p : players.values()) {
            if (p.role == Role.SEER && p.inspectTarget != null) {
                Player tgt = players.get(p.inspectTarget);
                sendPrivate(p.id, "\uD83D\uDD2E " + tgt.name + " は " +
                    (tgt.role == Role.WEREWOLF ? "人狼です" : "人狼ではありません"));
            }
        }

        for (Player p : players.values()) {
            if (p.role == Role.MEDIUM && lastExecuted != null) {
                Player ex = players.get(lastExecuted);
                sendPrivate(p.id, "\uD83D\uDC7B " + ex.name + " は " +
                    (ex.role == Role.WEREWOLF ? "人狼でした" : "人狼ではありませんでした"));
            }
        }

        phase = Phase.DISCUSS;
    }

    public void sendQuickReply(String userId, String message, List<String> options) {
        List<QuickReplyItem> items = options.stream().map(name ->
            QuickReplyItem.builder().action(
                MessageAction.withLabelAndText(name, "/select " + name)).build()
        ).collect(Collectors.toList());

        TextMessage msg = TextMessage.builder()
            .text(message)
            .quickReply(QuickReply.builder().items(items).build())
            .build();

        client.pushMessage(new PushMessage(userId, msg));
    }

    public List<String> getAlivePlayerDisplayNames(String exceptUserId) {
        return players.values().stream()
            .filter(p -> !p.id.equals(exceptUserId) && p.alive)
            .map(p -> p.name)
            .collect(Collectors.toList());
    }

    public void sendPrivate(String userId, String text) {
        client.pushMessage(new PushMessage(userId, new TextMessage(text)));
    }

    public void sendGroup(String text) {
        client.pushMessage(new PushMessage(groupId, new TextMessage(text)));
    }

    public void vote(String voterId, String targetId) {
        players.get(voterId).voteTarget = targetId;
    }

    public void executeVote() {
        Map<String, Long> tally = players.values().stream()
            .filter(p -> p.voteTarget != null)
            .collect(Collectors.groupingBy(p -> p.voteTarget, Collectors.counting()));

        String executed = Collections.max(tally.entrySet(), Map.Entry.comparingByValue()).getKey();
        Player p = players.get(executed);
        p.alive = false;
        lastExecuted = executed;

        sendGroup("⚖️ " + p.name + " が処刑されました。");

        phase = Phase.CHECK;
        checkVictory();
    }

    public void checkVictory() {
        long werewolves = players.values().stream().filter(p -> p.alive && p.role == Role.WEREWOLF).count();
        long villagers = players.values().stream().filter(p -> p.alive && p.role != Role.WEREWOLF).count();

        if (werewolves == 0) {
            sendGroup("\uD83C\uDF89 村人陣営の勝利！");
            phase = Phase.END;
        } else if (villagers <= werewolves) {
            sendGroup("\uD83D\uDC80 人狼陣営の勝利！");
            phase = Phase.END;
        } else {
            phase = Phase.NIGHT;
            sendGroup("\uD83C\uDF19 次の夜が訪れます。");
        }
    }
}

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
