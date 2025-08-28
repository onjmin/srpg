<script lang="ts">
    import { onDestroy } from "svelte";

    // --- 定数 ---
    const GRID_SIZE = 10;
    const PLAYER_STATS = { HP: 15, ATK: 3, MOVE: 3 };
    const DOG_STATS = { MOVE: 4, HEAL: 1 };
    const ENEMY_STATS = { HP: 6, ATK: 2, MOVE: 2 };
    const ATTACK_RANGE = 1;
    const METER_INCREASE_PER_TURN = 10;
    const MAX_METER = 100;
    const PENALTY_FACTOR = 0.5;
    const INITIAL_RATIONS = 50;
    const RATION_DECREASE_PER_TURN = 1;
    const INITIAL_CASTLES = [{ x: 5, y: 5, owner: "neutral" }];

    const CHARACTERS = {
        PLAYER: "👧",
        DOG: "🐕",
        ENEMY: "👻",
        TOILET: "🚽",
        BUSH: "🌳",
        GRASS: "🟩",
        CASTLE: "🏰",
    };

    // --- State ---
    let turn: "player" | "enemy" | "game_over" | "game_clear" = "player";
    let playerPos = { x: 0, y: 0 };
    let dogPos = { x: 1, y: 0 };
    let playerHP = PLAYER_STATS.HP;
    let peeMeter = 0;
    let poopMeter = 0;
    let rations = INITIAL_RATIONS;
    let message = "";
    let messageVisible = false;
    let enemies = [
        { id: 1, x: 8, y: 8, hp: ENEMY_STATS.HP, atk: ENEMY_STATS.ATK },
        { id: 2, x: 7, y: 3, hp: ENEMY_STATS.HP, atk: ENEMY_STATS.ATK },
    ];
    let reliefPoints = [
        { x: 3, y: 5, type: "TOILET" },
        { x: 8, y: 2, type: "BUSH" },
    ];
    let castles = [...INITIAL_CASTLES];
    let hasMoved = false;
    let isPlayerTurnActive = true;
    let selectedUnit: string | null = null;
    let validMoves: { x: number; y: number }[] = [];

    let timeoutId: ReturnType<typeof setTimeout> | null = null;
    onDestroy(() => {
        if (timeoutId) clearTimeout(timeoutId);
    });

    // --- effect: ターン進行 / 敵AI ---
    $: if (turn === "enemy") {
        isPlayerTurnActive = false;
        showMessage("敵のターン！");

        enemies = enemies.map((enemy) => {
            const newPos = { ...enemy };
            const distToPlayer =
                Math.abs(playerPos.x - enemy.x) +
                Math.abs(playerPos.y - enemy.y);
            const canAttack = distToPlayer <= ATTACK_RANGE;

            if (canAttack) {
                playerHP = Math.max(0, playerHP - enemy.atk);
                showMessage(`敵の攻撃！HPが ${enemy.atk} 減った...`);
            } else {
                let dx = playerPos.x - enemy.x;
                let dy = playerPos.y - enemy.y;
                let moveCount = 0;
                let cx = newPos.x;
                let cy = newPos.y;

                while (
                    moveCount < ENEMY_STATS.MOVE &&
                    (cx !== playerPos.x || cy !== playerPos.y)
                ) {
                    cx += dx > 0 ? 1 : dx < 0 ? -1 : 0;
                    cy += dy > 0 ? 1 : dy < 0 ? -1 : 0;
                    moveCount++;
                }
                newPos.x = cx;
                newPos.y = cy;
            }
            return { ...enemy, ...newPos };
        });

        timeoutId = setTimeout(() => {
            turn = "player";
            isPlayerTurnActive = true;
            hasMoved = false;

            const isInCastle = castles.some(
                (c) =>
                    c.x === playerPos.x &&
                    c.y === playerPos.y &&
                    c.owner === "player",
            );
            if (isInCastle) {
                playerHP = Math.min(
                    PLAYER_STATS.HP,
                    playerHP + DOG_STATS.HEAL + 2,
                );
                rations += 5;
                showMessage("お城でHPと兵糧が少し回復した！");
            } else {
                playerHP = Math.min(PLAYER_STATS.HP, playerHP + DOG_STATS.HEAL);
            }

            peeMeter = Math.min(peeMeter + METER_INCREASE_PER_TURN, MAX_METER);
            poopMeter = Math.min(
                poopMeter + METER_INCREASE_PER_TURN,
                MAX_METER,
            );
            rations = Math.max(0, rations - RATION_DECREASE_PER_TURN);

            showMessage("あなたのターン！");
        }, 1500);
    }

    // --- 関数群 ---
    function findValidMoves(pos: { x: number; y: number }, moveRange: number) {
        validMoves = [];
        for (let x = 0; x < GRID_SIZE; x++) {
            for (let y = 0; y < GRID_SIZE; y++) {
                const dist = Math.abs(x - pos.x) + Math.abs(y - pos.y);
                if (dist <= moveRange) validMoves.push({ x, y });
            }
        }
    }

    function handleCellClick(x: number, y: number) {
        if (!isPlayerTurnActive || hasMoved) return;
        const isValid = validMoves.some((m) => m.x === x && m.y === y);

        if (isValid) {
            const reliefPoint = reliefPoints.find(
                (p) => p.x === x && p.y === y,
            );
            if (reliefPoint) {
                if (peeMeter > 0 || poopMeter > 0) {
                    peeMeter = 0;
                    poopMeter = 0;
                    showMessage("スッキリした！");
                } else {
                    showMessage("特に用事はないかな...");
                }
                endPlayerTurn();
                return;
            }

            const oldPos = { ...playerPos };
            playerPos = { x, y };
            dogPos = oldPos;
            hasMoved = true;

            const castleOnTile = castles.find((c) => c.x === x && c.y === y);
            if (castleOnTile && castleOnTile.owner !== "player") {
                castles = castles.map((c) =>
                    c.x === x && c.y === y ? { ...c, owner: "player" } : c,
                );
                showMessage("お城を占領しました！");
            } else {
                showMessage("移動しました");
            }

            validMoves = [];
        } else {
            showMessage("そこには移動できません。");
        }
    }

    function handleUnitSelect(unit: string) {
        if (!isPlayerTurnActive || hasMoved) return;
        selectedUnit = unit;
        let moveRange = PLAYER_STATS.MOVE;

        if (peeMeter >= MAX_METER || poopMeter >= MAX_METER) {
            moveRange = Math.floor(moveRange * PENALTY_FACTOR);
            showMessage("尿意・便意で移動力が低下している！");
        }
        if (rations === 0) {
            moveRange = Math.floor(moveRange * PENALTY_FACTOR);
            showMessage("兵糧不足で移動力が低下している！");
        }

        findValidMoves(playerPos, moveRange);
        showMessage("移動先を選択してください。");
    }

    function handleAttack() {
        if (!isPlayerTurnActive || !hasMoved) {
            showMessage("まずは移動してから攻撃してください！");
            return;
        }

        const enemiesToHit = enemies.filter(
            (e) =>
                Math.abs(e.x - playerPos.x) <= ATTACK_RANGE &&
                Math.abs(e.y - playerPos.y) <= ATTACK_RANGE,
        );

        if (enemiesToHit.length > 0) {
            showMessage("攻撃！");
            enemies = enemies
                .map((enemy) => {
                    if (enemiesToHit.some((e) => e.id === enemy.id)) {
                        let atk = PLAYER_STATS.ATK;
                        if (
                            peeMeter >= MAX_METER ||
                            poopMeter >= MAX_METER ||
                            rations === 0
                        ) {
                            atk = Math.floor(atk * PENALTY_FACTOR);
                            showMessage(
                                "尿意・便意または兵糧不足で攻撃力が低下している！",
                            );
                        }
                        return { ...enemy, hp: enemy.hp - atk };
                    }
                    return enemy;
                })
                .filter((e) => e.hp > 0);
        } else {
            showMessage("攻撃できる敵がいません。");
        }

        endPlayerTurn();
    }

    function endPlayerTurn() {
        isPlayerTurnActive = false;
        showMessage("ターン終了");
        validMoves = [];
        selectedUnit = null;
        turn = "enemy";
    }

    function showMessage(msg: string) {
        message = msg;
        messageVisible = true;
        setTimeout(() => (messageVisible = false), 2000);
    }
</script>

<!-- --- UI --- -->
<div
    class="min-h-screen bg-gray-100 flex flex-col items-center justify-center p-4 font-sans"
>
    <div
        class="bg-white rounded-xl shadow-2xl p-6 md:p-8 w-full max-w-2xl text-center"
    >
        <h1 class="text-3xl md:text-4xl font-bold text-gray-800 mb-2">
            排泄管理SRPG
        </h1>
        <p class="text-sm md:text-base text-gray-500 mb-6">
            キャラクターをクリックして移動先を選ぼう！
        </p>

        <!-- 盤面 -->
        <div
            class="grid gap-0.5"
            style="grid-template-columns: repeat({GRID_SIZE}, 1fr);"
        >
            {#each Array(GRID_SIZE) as _, y}
                {#each Array(GRID_SIZE) as _, x}
                    {#if playerPos.x === x && playerPos.y === y}
                        <div
                            class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-indigo-300"
                            onclick={() => handleUnitSelect("player")}
                            tabindex="0"
                            role="button"
                            onkeydown={() => {}}
                        >
                            {CHARACTERS.PLAYER}
                        </div>
                    {:else if dogPos.x === x && dogPos.y === y}
                        <div
                            class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-amber-300"
                        >
                            {CHARACTERS.DOG}
                        </div>
                    {:else if enemies.some((e) => e.x === x && e.y === y)}
                        <div
                            class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-red-400"
                            onclick={() => handleCellClick(x, y)}
                            tabindex="0"
                            role="button"
                            onkeydown={() => {}}
                        >
                            {CHARACTERS.ENEMY}
                        </div>
                    {:else if castles.find((c) => c.x === x && c.y === y)}
                        {#each castles as c (c.x + "-" + c.y)}
                            {#if c.x === x && c.y === y}
                                <div
                                    class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center {c.owner ===
                                    'player'
                                        ? 'bg-blue-300'
                                        : 'bg-gray-400'}"
                                    onclick={() => handleCellClick(x, y)}
                                    tabindex="0"
                                    role="button"
                                    onkeydown={() => {}}
                                >
                                    {CHARACTERS.CASTLE}
                                </div>
                            {/if}
                        {/each}
                    {:else if reliefPoints.find((p) => p.x === x && p.y === y)}
                        {#each reliefPoints as p (p.x + "-" + p.y)}
                            {#if p.x === x && p.y === y}
                                <div
                                    class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-green-300"
                                    onclick={() => handleCellClick(x, y)}
                                    tabindex="0"
                                    role="button"
                                    onkeydown={() => {}}
                                >
                                    {p.type === "TOILET"
                                        ? CHARACTERS.TOILET
                                        : CHARACTERS.BUSH}
                                </div>
                            {/if}
                        {/each}
                    {:else if validMoves.some((m) => m.x === x && m.y === y)}
                        <div
                            class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-blue-300 opacity-70 border-blue-500"
                            onclick={() => handleCellClick(x, y)}
                            tabindex="0"
                            role="button"
                            onkeydown={() => {}}
                        >
                            {CHARACTERS.GRASS}
                        </div>
                    {:else}
                        <div
                            class="w-8 h-8 md:w-10 md:h-10 border flex items-center justify-center bg-green-50"
                            onclick={() => handleCellClick(x, y)}
                            tabindex="0"
                            role="button"
                            onkeydown={() => {}}
                        >
                            {CHARACTERS.GRASS}
                        </div>
                    {/if}
                {/each}
            {/each}
        </div>

        <!-- パネル -->
        <div class="mt-8 grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="bg-gray-100 p-4 rounded-lg shadow-inner">
                <h2 class="text-lg font-semibold text-gray-700 mb-2">
                    プレイヤー
                </h2>
                <p>HP: {playerHP}</p>
                <p>ターン: {turn === "player" ? "あなたの番" : "敵の番"}</p>
            </div>
            <div class="bg-gray-100 p-4 rounded-lg shadow-inner">
                <h2 class="text-lg font-semibold text-gray-700 mb-2">
                    ステータス
                </h2>
                <div class="mb-2">
                    <h3 class="text-sm font-medium">尿意</h3>
                    <div class="w-full bg-gray-300 rounded-full h-3">
                        <div
                            class="bg-indigo-500 rounded-full h-3"
                            style={`width:${peeMeter}%`}
                        ></div>
                    </div>
                </div>
                <div>
                    <h3 class="text-sm font-medium">便意</h3>
                    <div class="w-full bg-gray-300 rounded-full h-3">
                        <div
                            class="bg-orange-500 rounded-full h-3"
                            style={`width:${poopMeter}%`}
                        ></div>
                    </div>
                </div>
            </div>
        </div>

        <div class="mt-4 bg-gray-100 p-4 rounded-lg shadow-inner">
            <h2 class="text-lg font-semibold text-gray-700 mb-2">兵糧</h2>
            <div class="w-full bg-gray-300 rounded-full h-3">
                <div
                    class="bg-purple-500 rounded-full h-3"
                    style={`width:${rations}%`}
                ></div>
            </div>
            <p class="text-sm text-gray-500 mt-2">{rations}%</p>
        </div>

        <!-- ボタン -->
        <div class="mt-4 flex justify-center space-x-4">
            <button
                onclick={handleAttack}
                disabled={!isPlayerTurnActive || !hasMoved}
                class="px-4 py-2 bg-red-500 text-white font-bold rounded-lg shadow-md hover:bg-red-600 transition-colors duration-200 disabled:bg-gray-400"
            >
                攻撃
            </button>
            <button
                onclick={endPlayerTurn}
                disabled={!isPlayerTurnActive}
                class="px-4 py-2 bg-blue-500 text-white font-bold rounded-lg shadow-md hover:bg-blue-600 transition-colors duration-200 disabled:bg-gray-400"
            >
                ターン終了
            </button>
        </div>
    </div>

    {#if messageVisible}
        <div
            class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-yellow-200 text-yellow-800 p-4 rounded-lg shadow-2xl z-50 animate-pulse"
        >
            <p class="text-lg font-semibold">{message}</p>
        </div>
    {/if}
</div>
