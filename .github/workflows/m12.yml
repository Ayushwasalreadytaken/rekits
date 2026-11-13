package com.xyron.kits;

import org.bukkit.Bukkit;
import org.bukkit.ChatColor;
import org.bukkit.Material;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.command.TabExecutor;
import org.bukkit.configuration.ConfigurationSection;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Entity;
import org.bukkit.entity.Player;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.PlayerInventory;
import org.bukkit.plugin.java.JavaPlugin;

import java.io.File;
import java.io.IOException;
import java.util.*;

/**
 * KitsPlugin - works on Spigot/Paper 1.13 up to the latest version (incl. 1.21.x), Java 8+.
 */
public class KitsPlugin extends JavaPlugin implements TabExecutor {

    private File file;
    private YamlConfiguration data;

    @Override
    public void onEnable() {
        getDataFolder().mkdirs();
        file = new File(getDataFolder(), "kits.yml");
        data = YamlConfiguration.loadConfiguration(file);
        getCommand("kit").setExecutor(this);
        getCommand("kit").setTabCompleter(this);
    }

    private void save() {
        try {
            data.save(file);
        } catch (IOException e) {
            getLogger().severe("Could not save kits.yml: " + e.getMessage());
        }
    }

    private void msg(CommandSender s, String m) {
        s.sendMessage(ChatColor.GOLD + "[Kits] " + ChatColor.YELLOW + m);
    }

    private Set<String> kitNames() {
        ConfigurationSection s = data.getConfigurationSection("kits");
        return s == null ? new TreeSet<String>() : new TreeSet<String>(s.getKeys(false));
    }

    private String findKit(String name) {
        if (name == null) return null;
        for (String k : kitNames()) if (k.equalsIgnoreCase(name)) return k;
        return null;
    }

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (!sender.hasPermission("kits.admin")) {
            msg(sender, "You don't have permission.");
            return true;
        }
        if (args.length == 0) {
            msg(sender, "/kit create <name> | saved | take <name> | delete <name>");
            msg(sender, "/kit give [name] <@a|@p|@r|player...> [clear] [except <player...>]");
            return true;
        }
        String sub = args[0].toLowerCase();
        if (sub.equals("create")) create(sender, args);
        else if (sub.equals("saved") || sub.equals("list")) saved(sender);
        else if (sub.equals("take")) take(sender, args);
        else if (sub.equals("delete") || sub.equals("remove")) delete(sender, args);
        else if (sub.equals("give")) give(sender, args);
        else msg(sender, "Unknown subcommand. Use create, saved, take, give, delete.");
        return true;
    }

    // /kit create <name>  -> saves your current inventory
    private void create(CommandSender sender, String[] args) {
        if (!(sender instanceof Player)) { msg(sender, "Players only."); return; }
        Player p = (Player) sender;
        if (args.length < 2) { msg(sender, "Usage: /kit create <name>"); return; }
        String name = args[1].toLowerCase();
        String existing = findKit(name);
        if (existing != null) data.set("kits." + existing, null);

        String base = "kits." + name;
        PlayerInventory inv = p.getInventory();
        int count = 0;
        ItemStack[] storage = inv.getStorageContents();
        for (int i = 0; i < storage.length; i++) {
            if (valid(storage[i])) { data.set(base + ".items." + i, storage[i]); count++; }
        }
        ItemStack[] armor = inv.getArmorContents();
        for (int i = 0; i < armor.length; i++) {
            if (valid(armor[i])) { data.set(base + ".armor." + i, armor[i]); count++; }
        }
        if (valid(inv.getItemInOffHand())) { data.set(base + ".offhand", inv.getItemInOffHand()); count++; }
        if (count == 0) { msg(sender, "Your inventory is empty, nothing to save."); return; }
        data.set("last", name);
        save();
        msg(sender, "Kit '" + name + "' saved with " + count + " item stack(s)" + (existing != null ? " (replaced old one)." : "."));
    }

    private boolean valid(ItemStack i) {
        return i != null && i.getType() != Material.AIR;
    }

    // /kit saved
    private void saved(CommandSender sender) {
        Set<String> names = kitNames();
        if (names.isEmpty()) { msg(sender, "No kits saved yet. Use /kit create <name>."); return; }
        msg(sender, "Saved kits (" + names.size() + "): " + join(names));
    }

    private String join(Collection<String> c) {
        StringBuilder sb = new StringBuilder();
        for (String s : c) { if (sb.length() > 0) sb.append(", "); sb.append(s); }
        return sb.toString();
    }

    // /kit take <name>  -> gives the kit to yourself
    private void take(CommandSender sender, String[] args) {
        if (!(sender instanceof Player)) { msg(sender, "Players only."); return; }
        if (args.length < 2) { msg(sender, "Usage: /kit take <name>"); return; }
        String kit = findKit(args[1]);
        if (kit == null) { msg(sender, "No kit named '" + args[1] + "'."); return; }
        giveKit((Player) sender, kit);
        msg(sender, "You took kit '" + kit + "'.");
    }

    private void delete(CommandSender sender, String[] args) {
        if (args.length < 2) { msg(sender, "Usage: /kit delete <name>"); return; }
        String kit = findKit(args[1]);
        if (kit == null) { msg(sender, "No kit named '" + args[1] + "'."); return; }
        data.set("kits." + kit, null);
        save();
        msg(sender, "Deleted kit '" + kit + "'.");
    }

    // /kit give [kit] <targets...> [except <names...>]
    private void give(CommandSender sender, String[] args) {
        if (args.length < 2) { msg(sender, "Usage: /kit give [kit] <@a|@p|@r|player...> [except <player...>]"); return; }
        String kit;
        int start;
        boolean noKitGiven = args[1].startsWith("@") || isExcept(args[1])
                || (findKit(args[1]) == null && Bukkit.getPlayerExact(args[1]) != null);
        if (noKitGiven) {
            kit = findKit(data.getString("last", ""));
            if (kit == null) { msg(sender, "No default kit yet. Create one first with /kit create <name>."); return; }
            start = 1;
        } else {
            kit = findKit(args[1]);
            if (kit == null) { msg(sender, "No kit named '" + args[1] + "'."); return; }
            start = 2;
        }

        Set<Player> targets = new LinkedHashSet<Player>();
        Set<String> excluded = new HashSet<String>();
        boolean afterExcept = false;
        boolean clear = false;

        for (int i = start; i < args.length; i++) {
            String a = args[i];
            if (isExcept(a)) { afterExcept = true; continue; }
            if (a.equalsIgnoreCase("clear") && Bukkit.getPlayerExact(a) == null) { clear = true; continue; }
            if (a.startsWith("@")) {
                List<Player> found = selectPlayers(sender, a);
                if (found == null) { msg(sender, "Bad selector: " + a); return; }
                if (afterExcept) {
                    for (Player ep : found) excluded.add(ep.getName().toLowerCase());
                } else {
                    targets.addAll(found);
                }
            } else if (afterExcept) {
                excluded.add(a.toLowerCase());
            } else {
                Player tp = Bukkit.getPlayerExact(a);
                if (tp == null) { msg(sender, "Player '" + a + "' is not online. Nothing was given. (Use the word: except)"); return; }
                targets.add(tp);
            }
        }

        Iterator<Player> it = targets.iterator();
        while (it.hasNext()) {
            if (excluded.contains(it.next().getName().toLowerCase())) it.remove();
        }
        if (targets.isEmpty()) { msg(sender, "No players to give the kit to."); return; }

        int n = 0;
        for (Player p : targets) {
            giveKit(p, kit, clear);
            p.sendMessage(ChatColor.GOLD + "[Kits] " + ChatColor.YELLOW + "You received kit '" + kit + "'.");
            n++;
        }
        String skipped = excluded.isEmpty() ? "" : " (except " + join(excluded) + ")";
        msg(sender, "Gave kit '" + kit + "' to " + n + " player(s)" + skipped + ".");
    }


    /** Accepts "except" and common typos (excpt, exept, execpt...), unless it is a real online player's name. */
    private boolean isExcept(String a) {
        String l = a.toLowerCase();
        if (Bukkit.getPlayerExact(a) != null) return false;
        if (l.equals("except") || l.equals("not") || l.equals("without") || l.equals("but") || l.equals("-")) return true;
        return l.length() >= 4 && l.charAt(0) == 'e' && distance(l, "except") <= 2;
    }

    private int distance(String a, String b) {
        int[][] d = new int[a.length() + 1][b.length() + 1];
        for (int i = 0; i <= a.length(); i++) d[i][0] = i;
        for (int j = 0; j <= b.length(); j++) d[0][j] = j;
        for (int i = 1; i <= a.length(); i++)
            for (int j = 1; j <= b.length(); j++)
                d[i][j] = Math.min(Math.min(d[i - 1][j] + 1, d[i][j - 1] + 1),
                        d[i - 1][j - 1] + (a.charAt(i - 1) == b.charAt(j - 1) ? 0 : 1));
        return d[a.length()][b.length()];
    }

    /** Resolves @a, @p, @r, @s (and any other vanilla selector). Returns null if invalid. */
    private List<Player> selectPlayers(CommandSender sender, String sel) {
        List<Player> out = new ArrayList<Player>();
        try {
            for (Entity e : Bukkit.selectEntities(sender, sel)) {
                if (e instanceof Player) out.add((Player) e);
            }
            return out;
        } catch (Throwable t) {
            // Fallback for sources where vanilla selectors are not available (e.g. console @p)
            String s = sel.toLowerCase();
            if (s.equals("@a")) { out.addAll(Bukkit.getOnlinePlayers()); return out; }
            if (s.equals("@s") && sender instanceof Player) { out.add((Player) sender); return out; }
            if (s.equals("@r")) {
                List<Player> all = new ArrayList<Player>(Bukkit.getOnlinePlayers());
                if (!all.isEmpty()) out.add(all.get(new Random().nextInt(all.size())));
                return out;
            }
            return null;
        }
    }

    private void giveKit(Player p, String kit) { giveKit(p, kit, false); }

    private void giveKit(Player p, String kit, boolean clear) {
        ConfigurationSection s = data.getConfigurationSection("kits." + kit);
        if (s == null) return;
        PlayerInventory inv = p.getInventory();
        if (clear) {
            inv.clear();
            inv.setArmorContents(new ItemStack[4]);
            inv.setItemInOffHand(null);
        }
        List<ItemStack> extra = new ArrayList<ItemStack>();

        ItemStack[] cur = inv.getArmorContents();
        for (int i = 0; i < 4 && i < cur.length; i++) {
            ItemStack a = s.getItemStack("armor." + i);
            if (a == null) continue;
            if (!valid(cur[i])) cur[i] = a.clone();
            else extra.add(a.clone());
        }
        inv.setArmorContents(cur);

        ItemStack off = s.getItemStack("offhand");
        if (off != null) {
            if (!valid(inv.getItemInOffHand())) inv.setItemInOffHand(off.clone());
            else extra.add(off.clone());
        }

        ConfigurationSection items = s.getConfigurationSection("items");
        if (items != null) {
            for (String k : items.getKeys(false)) {
                ItemStack it = items.getItemStack(k);
                if (it == null) continue;
                int slot = -1;
                try { slot = Integer.parseInt(k); } catch (NumberFormatException ignored) { }
                // keep the kit's original slot when it is free (hotbar stays hotbar)
                if (slot >= 0 && slot < 36 && !valid(inv.getItem(slot))) inv.setItem(slot, it.clone());
                else extra.add(it.clone());
            }
        }

        Map<Integer, ItemStack> left = inv.addItem(extra.toArray(new ItemStack[0]));
        for (ItemStack drop : left.values()) p.getWorld().dropItemNaturally(p.getLocation(), drop);
    }

    @Override
    public List<String> onTabComplete(CommandSender sender, Command cmd, String alias, String[] args) {
        if (!sender.hasPermission("kits.admin")) return new ArrayList<String>();
        List<String> out = new ArrayList<String>();
        String first = args[0].toLowerCase();
        if (args.length == 1) {
            out.addAll(Arrays.asList("create", "saved", "take", "give", "delete"));
        } else if (args.length == 2 && (first.equals("take") || first.equals("give") || first.equals("delete"))) {
            out.addAll(kitNames());
            if (first.equals("give")) out.addAll(Arrays.asList("@a", "@p", "@r", "@s"));
        } else if (first.equals("give") && args.length >= 3) {
            boolean afterExcept = false;
            for (String a : args) if (a.equalsIgnoreCase("except")) afterExcept = true;
            if (!afterExcept) out.addAll(Arrays.asList("@a", "@p", "@r", "@s", "clear", "except"));
            for (Player p : Bukkit.getOnlinePlayers()) out.add(p.getName());
        }
        String last = args[args.length - 1].toLowerCase();
        Iterator<String> it = out.iterator();
        while (it.hasNext()) if (!it.next().toLowerCase().startsWith(last)) it.remove();
        return out;
    }
}
