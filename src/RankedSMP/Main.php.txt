<?php

namespace RankedSMP;

use pocketmine\plugin\PluginBase;
use pocketmine\event\Listener;
use pocketmine\utils\Config;
use pocketmine\player\Player;
use pocketmine\command\Command;
use pocketmine\command\CommandSender;

use pocketmine\event\player\PlayerJoinEvent;
use pocketmine\event\player\PlayerDeathEvent;
use pocketmine\event\entity\EntityDamageByEntityEvent;
use pocketmine\event\entity\EntityEffectAddEvent;

class Main extends PluginBase implements Listener {

    private Config $ranks;
    private bool $rankStartPending = false;

    public function onEnable(): void {
        @mkdir($this->getDataFolder());
        $this->ranks = new Config($this->getDataFolder() . "ranks.yml", Config::YAML);
        $this->getServer()->getPluginManager()->registerEvents($this, $this);
    }

    /* ===================== RANK STORAGE ===================== */

    public function getRank(Player $player): ?int {
        return $this->ranks->get($player->getName(), null);
    }

    public function setRank(Player $player, int $rank): void {
        $this->ranks->set($player->getName(), $rank);
        $this->ranks->save();
        $this->applyRankHealth($player);
    }

    public function clearRanks(): void {
        $this->ranks->setAll([]);
        $this->ranks->save();
    }

    /* ===================== HEALTH SCALING ===================== */

    public function applyRankHealth(Player $player): void {
        $rank = $this->getRank($player);

        if ($rank === null) {
            $player->setMaxHealth(20);
            if ($player->getHealth() > 20) {
                $player->setHealth(20);
            }
            return;
        }

        // +1 HP (half-heart) per rank closer to #1
        $extraHp = 21 - $rank;
        $maxHealth = 20 + $extraHp;

        $player->setMaxHealth($maxHealth);
        if ($player->getHealth() > $maxHealth) {
            $player->setHealth($maxHealth);
        }
    }

    public function onJoin(PlayerJoinEvent $event): void {
        $this->applyRankHealth($event->getPlayer());
    }

    /* ===================== RANK SWAP ON KILL ===================== */

    public function onDeath(PlayerDeathEvent $event): void {
        $victim = $event->getPlayer();
        $cause = $victim->getLastDamageCause();

        if (!$cause instanceof EntityDamageByEntityEvent) return;

        $killer = $cause->getDamager();
        if (!$killer instanceof Player) return;

        $victimRank = $this->getRank($victim);
        $killerRank = $this->getRank($killer);

        if ($victimRank === null || $killerRank === null) return;
        if ($killerRank <= $victimRank) return;

        // swap ranks
        $this->setRank($killer, $victimRank);
        $this->setRank($victim, $killerRank);

        $killer->sendMessage("§aYou stole Rank §e#$victimRank§a!");
        $victim->sendMessage("§cYou were demoted to Rank §e#$killerRank§c.");
    }

    /* ===================== POTION SCALING ===================== */

    public function onEffectAdd(EntityEffectAddEvent $event): void {
        $entity = $event->getEntity();
        if (!$entity instanceof Player) return;

        $rank = $this->getRank($entity);
        if ($rank === null) return;

        // +5 seconds per rank closer to #1
        $extraSeconds = (21 - $rank) * 5;
        $extraTicks = $extraSeconds * 20;

        $effect = $event->getEffect();
        $effect->setDuration($effect->getDuration() + $extraTicks);
        $event->setEffect($effect);
    }

    /* ===================== COMMANDS ===================== */

    public function onCommand(CommandSender $sender, Command $command, string $label, array $args): bool {
        switch ($command->getName()) {

            case "rank":
                if (!$sender instanceof Player) return true;
                $rank = $this->getRank($sender);
                $sender->sendMessage(
                    $rank === null ? "§7You are unranked." : "§6Your Rank: §e#$rank"
                );
                return true;

            case "ranks":
                $all = $this->ranks->getAll();
                $map = [];
                foreach ($all as $name => $rank) {
                    $map[(int)$rank] = $name;
                }

                $sender->sendMessage("§6§l=== Ranked SMP Ranks ===");
                for ($i = 1; $i <= 20; $i++) {
                    $sender->sendMessage(
                        isset($map[$i])
                        ? "§e#$i §7→ §a{$map[$i]}"
                        : "§e#$i §7→ §8Unclaimed"
                    );
                }
                return true;

            case "setrank":
                if (!$sender->hasPermission("rank.admin")) return true;
                if (count($args) !== 2) {
                    $sender->sendMessage("/setrank <player> <1-20>");
                    return true;
                }

                $player = $this->getServer()->getPlayerExact($args[0]);
                if (!$player) {
                    $sender->sendMessage("Player not online.");
                    return true;
                }

                $rank = (int)$args[1];
                if ($rank < 1 || $rank > 20) {
                    $sender->sendMessage("Rank must be between 1 and 20.");
                    return true;
                }

                $this->setRank($player, $rank);
                $sender->sendMessage("§aSet {$player->getName()} to Rank #$rank");
                return true;

            case "rankstart":
                if (!$sender->hasPermission("rank.admin")) return true;

                if (!isset($args[0]) || $args[0] !== "confirm") {
                    $this->rankStartPending = true;
                    $sender->sendMessage("§c§lWARNING!");
                    $sender->sendMessage("§7This will reset ALL ranks.");
                    $sender->sendMessage("§eType §6/rankstart confirm §eto continue.");
                    return true;
                }

                if (!$this->rankStartPending) {
                    $sender->sendMessage("§cNo rank start pending.");
                    return true;
                }

                $players = array_values($this->getServer()->getOnlinePlayers());
                $this->clearRanks();

                shuffle($players);
                $players = array_slice($players, 0, 20);

                $ranks = range(1, count($players));
                shuffle($ranks);

                foreach ($players as $i => $player) {
                    $this->setRank($player, $ranks[$i]);
                    $player->sendMessage("§6You were assigned §eRank #{$ranks[$i]}");
                }

                $this->rankStartPending = false;
                $this->getServer()->broadcastMessage("§6§lRanked SMP has begun!");
                return true;
        }
        return false;
    }
}
