package be.drsoulz.hub;

import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.Location;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.player.PlayerCommandSendEvent;
import org.bukkit.event.player.PlayerJumpEvent;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.event.player.PlayerQuitEvent;
import org.bukkit.Sound;

import java.util.HashMap;
import java.util.HashSet;
import java.util.UUID;

public class Main extends JavaPlugin implements CommandExecutor, Listener {
    private Location hubLocation;
    private HashMap<UUID, Location> lastKnownLocations = new HashMap<>();
    private HashSet<UUID> doubleJumpCooldown = new HashSet<>();

    @Override
    public void onEnable() {
        // Enregistrez votre plugin et définissez l'exécuteur de commandes
        this.getCommand("sethub").setExecutor(this);
        this.getCommand("hub").setExecutor(this);

        // Enregistrez le gestionnaire d'événements
        getServer().getPluginManager().registerEvents(this, this);
    }

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (cmd.getName().equalsIgnoreCase("sethub")) {
            if (sender instanceof Player) {
                Player player = (Player) sender;
                // Obtenez la position actuelle du joueur et définissez-la comme emplacement du hub
                hubLocation = player.getLocation();
                player.sendMessage("Le hub a été défini avec succès !");
                return true;
            } else {
                sender.sendMessage("Cette commande doit être exécutée par un joueur !");
                return false;
            }
        }

        if (cmd.getName().equalsIgnoreCase("hub")) {
            if (sender instanceof Player) {
                Player player = (Player) sender;

                // Ajoutez ici les commandes pour quitter les mini-jeux, en utilisant le symbole '/' pour éviter les messages de confirmation
                player.performCommand("/tr leave");
                player.performCommand("/pb leave");
                player.performCommand("/bb leave");

                // Téléportez le joueur à l'emplacement du hub s'il a été défini
                if (hubLocation != null) {
                    player.teleport(hubLocation);
                    return true;
                } else {
                    player.sendMessage("Le hub n'a pas été défini. Utilisez /sethub pour le définir !");
                    return false;
                }
            } else {
                sender.sendMessage("Cette commande doit être exécutée par un joueur !");
                return false;
            }
        }

        return false;
    }

    // Gestionnaire d'événements pour bloquer les messages de confirmation lorsqu'un joueur envoie des commandes
    @EventHandler
    public void onPlayerCommandSend(PlayerCommandSendEvent event) {
        Player player = event.getPlayer();
        if (player.hasPermission("yourplugin.bypassleave")) {
            // Permettre aux joueurs avec la permission "yourplugin.bypassleave" de voir les messages de confirmation
            return;
        }

        // Bloquer les messages de confirmation des commandes pour quitter les mini-jeux
        event.getCommands().remove("tr leave");
        event.getCommands().remove("pb leave");
        event.getCommands().remove("bb leave");
    }

    // Gestionnaire d'événements pour le double jump
    @EventHandler
    public void onPlayerJump(PlayerJumpEvent event) {
        Player player = event.getPlayer();
        if (player.getWorld().getName().equals("world")) {
            // Si le joueur est dans le monde "world", permettez-lui de faire un double jump
            if (!doubleJumpCooldown.contains(player.getUniqueId())) {
                player.setVelocity(player.getVelocity().add(player.getLocation().getDirection().multiply(1.5).setY(1)));
                doubleJumpCooldown.add(player.getUniqueId());

                // Jouez un son lorsque le joueur effectue un double saut
                player.playSound(player.getLocation(), Sound.ENTITY_FIREWORK_ROCKET_LAUNCH, 1.0f, 1.0f);

                // Supprimez le joueur de l'ensemble après un court laps de temps pour le double jump
                getServer().getScheduler().runTaskLater(this, () -> doubleJumpCooldown.remove(player.getUniqueId()), 20L);
            }
        }
    }

    // Gestionnaire d'événements pour stocker la dernière position du joueur avant qu'il ne quitte le serveur
    @EventHandler
    public void onPlayerQuit(PlayerQuitEvent event) {
        Player player = event.getPlayer();
        lastKnownLocations.put(player.getUniqueId(), player.getLocation());
    }

    // Gestionnaire d'événements pour téléporter le joueur au hub lorsqu'il rejoint à nouveau le serveur
    @EventHandler
    public void onPlayerJoin(PlayerJoinEvent event) {
        Player player = event.getPlayer();
        if (hubLocation != null) {
            Location lastLocation = lastKnownLocations.get(player.getUniqueId());
            if (lastLocation != null) {
                player.teleport(hubLocation);
            }
        }
    }
}
