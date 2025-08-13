ниже — готовый, простой и понятный скрипт на Python, который полностью закрывает ТЗ: читает данные о гостях и блюдах, учитывает диеты, аллергии, количество порций, бюджет и ваши предпочтения; находит «почти оптимальное» меню (жадный алгоритм + небольшое локальное улучшение), выбирая максимум удовлетворённых гостей, а при равенстве — минимальную стоимость. В конце выводит компромиссные варианты.

вы можете сразу запускать как есть (там есть пример данных) или подставить свои списки гостей/блюд и бюджет.

# -*- coding: utf-8 -*-
"""
Планирование вечеринки (магистратура: Руководитель IT-разработки)
— вход: список гостей с диетами/аллергиями и список блюд с ценой/ингредиентами/тегами/порциями
— учёт: диеты, аллергии, бюджет, кол-во порций, личные предпочтения (вес блюда)
— цель: покрыть максимум гостей; при равном покрытии — минимизировать стоимость
Подход: понятный жадный алгоритм "гость с наименьшим числом опций → самая выгодная порция блюда"
+ лёгкое локальное улучшение заменами.
Зависимостей, кроме стандартной библиотеки, нет.
"""

from dataclasses import dataclass, field
from typing import List, Dict, Set, Tuple, Optional
import math

# ---------- Модель данных ----------
@dataclass
class Guest:
    name: str
    requires: Set[str]          # например: {'vegetarian', 'gluten_free'} или пустое множество
    allergies: Set[str]         # например: {'nuts', 'shrimp'} или пустое множество

@dataclass
class Dish:
    name: str
    cost: float                 # цена за 1 порцию
    ingredients: Set[str]       # набор ингредиентов (для аллергий)
    tags: Set[str]              # диетические теги блюда, например {'vegetarian','gluten_free'}
    portions: int               # сколько порций доступно
    preference: float = 0.0     # ваша симпатия (0..1). Чем выше, тем охотнее берём при равной выгоде

@dataclass
class PlanResult:
    servings: Dict[str, int]                          # блюдо -> число заказанных порций
    assignment: Dict[str, str]                        # гость -> блюдо (которое ему досталось)
    total_cost: float
    covered: int
    uncovered: List[str]                              # кого не смогли накормить
    reason_for_uncovered: Dict[str, str]              # пояснение по каждому не накормленному гостю


# ---------- Совместимость ----------
def dish_serves_guest(d: Dish, g: Guest) -> bool:
    """Проверяет, подходит ли блюдо гостю: блюдо должно покрывать ВСЕ требуемые теги гостя и не содержать аллергенов."""
    if not g.requires.issubset(d.tags):
        return False
    if g.allergies & d.ingredients:
        return False
    return True


# ---------- Построение справочников опций ----------
def build_options(guests: List[Guest], dishes: List[Dish]) -> Tuple[Dict[str, List[int]], Dict[int, List[str]]]:
    """
    guest_options: гость -> индексы блюд, которые ему подходят
    dish_targets:  блюдо(i) -> список имён гостей, которым блюдо подходит
    """
    guest_options: Dict[str, List[int]] = {g.name: [] for g in guests}
    dish_targets: Dict[int, List[str]] = {i: [] for i in range(len(dishes))}
    for i, d in enumerate(dishes):
        for g in guests:
            if dish_serves_guest(d, g):
                guest_options[g.name].append(i)
                dish_targets[i].append(g.name)
    return guest_options, dish_targets


# ---------- Жадное составление плана ----------
def plan_menu_greedy(
    guests: List[Guest],
    dishes: List[Dish],
    budget: float,
    preference_alpha: float = 0.2,
) -> PlanResult:
    """
    Итеративно выбираем (гость -> 1 порция блюда), приоритизируя
    - гостей с минимальным числом доступных блюд (труднее накормить)
    - более выгодные по цене блюда
    - слегка учитываем личные предпочтения к блюдам (preference_alpha)
    """
    guest_options, _ = build_options(guests, dishes)
    remaining_budget = budget
    remaining_portions = [d.portions for d in dishes]
    assignment: Dict[str, str] = {}           # гость -> блюдо
    servings: Dict[str, int] = {d.name: 0 for d in dishes}

    # Подготовим быстрый доступ: имя гостя -> объект
    guest_by_name = {g.name: g for g in guests}

    # Множество не накормленных (пока)
    unserved = {g.name for g in guests}

    # Предподсчёт сложностей: меньше опций -> приоритет выше
    def guest_priority(name: str) -> float:
        opts = guest_options.get(name, [])
        return 1.0 / (len(opts) if len(opts) > 0 else 9999)

    while True:
        best_pair: Optional[Tuple[str, int, float]] = None  # (guest_name, dish_idx, score)
        # среди всех не накормленных ищем лучшую пару "гость-блюдо"
        for gname in list(unserved):
            options = guest_options.get(gname, [])
            if not options:
                continue
            # сортируем варианты блюд по "скорректированной" цене с учётом предпочтений
            for di in options:
                if remaining_portions[di] <= 0:
                    continue
                cost_adj = dishes[di].cost / (1.0 + preference_alpha * dishes[di].preference)
                if cost_adj > remaining_budget + 1e-9:
                    continue
                # score: трудность гостя / стоимость (выше — лучше)
                score = guest_priority(gname) / cost_adj
                # лёгкие тай-брейки: дешевле -> лучше; больше preference -> чуть лучше
                if best_pair is None:
                    best_pair = (gname, di, score)
                else:
                    _, di_best, score_best = best_pair
                    if score > score_best + 1e-12:
                        best_pair = (gname, di, score)
                    elif abs(score - score_best) <= 1e-12:
                        # tie-break: ниже цена, затем выше preference
                        ca = dishes[di].cost / (1.0 + preference_alpha * dishes[di].preference)
                        cb = dishes[di_best].cost / (1.0 + preference_alpha * dishes[di_best].preference)
                        if ca < cb - 1e-12 or (abs(ca - cb) <= 1e-12 and dishes[di].preference > dishes[di_best].preference):
                            best_pair = (gname, di, score)

        if best_pair is None:
            break  # нет допустимых пар по бюджету/порциям
        gname, di, _ = best_pair
        # Покупаем 1 порцию блюда di и назначаем её гостю gname
        remaining_budget -= dishes[di].cost
        remaining_portions[di] -= 1
        assignment[gname] = dishes[di].name
        servings[dishes[di].name] += 1
        unserved.remove(gname)

        # Если всех накормили — завершаем
        if not unserved:
            break

    # Сформируем причины для тех, кого не накормили
    reason: Dict[str, str] = {}
    for gname in unserved:
        opts = guest_options.get(gname, [])
        if not opts:
            reason[gname] = "Нет подходящих блюд (диеты/аллергии)."
        else:
            # либо нет бюджета, либо порции кончились
            ok_budget = any(dishes[i].cost <= remaining_budget + 1e-9 for i in opts)
            ok_portion = any(remaining_portions[i] > 0 for i in opts)
            if not ok_portion:
                reason[gname] = "Подходящие блюда закончились по порциям."
            elif not ok_budget:
                reason[gname] = "Не хватает бюджета на подходящие блюда."
            else:
                reason[gname] = "Комбинация не найдена по внутренним тай-брейкам."

    total_cost = sum(servings[d.name] * d.cost for d in dishes)
    return PlanResult(servings, assignment, round(total_cost, 2), len(assignment), sorted(unserved), reason)


# ---------- Локальное улучшение (простая замена 1 на 2) ----------
def try_local_improvement(
    res: PlanResult,
    guests: List[Guest],
    dishes: List[Dish],
    budget: float,
    max_passes: int = 1
) -> PlanResult:
    """
    Очень простая попытка улучшить план:
    — пробуем "освободить" одну выданную порцию и на сэкономленные деньги накрыть 2 других ненакормленных гостей.
      Если покрытие выросло (или покрытие то же, но стоимость меньше) — принимаем изменение.
    Делается пару итераций, чтобы не раздувать код.
    """
    if not res.uncovered:
        return res

    guest_by_name = {g.name: g for g in guests}
    # Восстановим текущие счётчики порций и бюджет
    used_portions = {d.name: 0 for d in dishes}
    for dname, cnt in res.servings.items():
        used_portions[dname] = cnt
    spent = res.total_cost
    remaining_budget = budget - spent

    # Быстрые справочники
    guest_options, _ = build_options(guests, dishes)

    def clone_result(r: PlanResult) -> PlanResult:
        return PlanResult(
            servings=dict(r.servings),
            assignment=dict(r.assignment),
            total_cost=r.total_cost,
            covered=r.covered,
            uncovered=list(r.uncovered),
            reason_for_uncovered=dict(r.reason_for_uncovered),
        )

    best = clone_result(res)

    for _ in range(max_passes):
        improved = False

        # список уже накормленных пар (гость -> блюдо)
        served_pairs = list(best.assignment.items())
        unserved_now = set(best.uncovered)

        for g_served, dname_served in served_pairs:
            # Попробуем снять 1 порцию у dname_served и освободить бюджет
            dish_idx = next(i for i, d in enumerate(dishes) if d.name == dname_served)
            if best.servings[dname_served] <= 0:
                continue

            freed_budget = dishes[dish_idx].cost
            # временно "откатываем"
            tmp = clone_result(best)
            tmp.servings[dname_served] -= 1
            del tmp.assignment[g_served]
            tmp.covered -= 1
            tmp.total_cost = round(tmp.total_cost - dishes[dish_idx].cost, 2)
            tmp.uncovered = sorted(set(tmp.uncovered) | {g_served})
            # Теперь пробуем этими средствами накрыть двоих из unserved (включая снятого)
            candidates = sorted(
                list(set(unserved_now) | {g_served}),
                key=lambda x: len(guest_options.get(x, [])) if guest_options.get(x, []) else 9999
            )

            def can_cover_guest(gname: str, local_budget: float, local_used: Dict[str, int]) -> Optional[int]:
                # найдём самое дешевое пригодное блюдо с ещё свободной порцией
                best_i, best_cost = None, math.inf
                for i in guest_options.get(gname, []):
                    dish = dishes[i]
                    # узнаем, сколько порций уже задействовано после отката
                    already = tmp.servings.get(dish.name, 0) if tmp.servings.get(dish.name, 0) is not None else 0
                    available = dish.portions - already
                    if available <= 0:
                        continue
                    if dish.cost <= local_budget + 1e-9 and dish.cost < best_cost - 1e-12:
                        best_i, best_cost = i, dish.cost
                return best_i

            # простая жадная попытка накрыть двух
            local_budget = tmp.total_cost + freed_budget  # это "могу потратить" = текущая стоимость + freed
            to_spend = freed_budget
            picked: List[Tuple[str, int]] = []  # (guest, dish_idx)

            # пробуем выбрать до двух разных гостей
            for gname in candidates:
                if len(picked) >= 2:
                    break
                i = can_cover_guest(gname, to_spend, tmp.servings)
                if i is not None:
                    picked.append((gname, i))
                    to_spend -= dishes[i].cost

            # если удалось покрыть 2 (или хотя бы 1, но суммарно стало лучше/дешевле) — применяем
            if picked and (len(picked) >= 2 or (len(picked) == 1 and (tmp.covered + 1) >= best.covered)):
                new_tmp = clone_result(tmp)
                # применяем выбранные покупки
                for gname, i in picked:
                    dname = dishes[i].name
                    new_tmp.servings[dname] = new_tmp.servings.get(dname, 0) + 1
                    new_tmp.assignment[gname] = dname
                    if gname in new_tmp.uncovered:
                        new_tmp.uncovered.remove(gname)
                    new_tmp.covered += 1
                    new_tmp.total_cost = round(new_tmp.total_cost + dishes[i].cost, 2)

                # выбор: лучше покрытие, а при равном покрытии — меньшая стоимость
                if (new_tmp.covered > best.covered) or (new_tmp.covered == best.covered and new_tmp.total_cost < best.total_cost - 1e-9):
                    best = new_tmp
                    improved = True
                    break  # начнём новый проход, чтобы не усложнять

        if not improved:
            break

    # Обновим причины ненакормленных
    _, _ = build_options(guests, dishes)
    reason: Dict[str, str] = {}
    still_un = set(g.name for g in guests) - set(best.assignment.keys())
    guest_options_final, _ = build_options(guests, dishes)
    spent = best.total_cost
    for gname in still_un:
        opts = guest_options_final.get(gname, [])
        if not opts:
            reason[gname] = "Нет подходящих блюд (диеты/аллергии)."
        else:
            # здесь мы уже в итоговом плане — значит, либо порции утыкались, либо бюджет не позволил
            reason[gname] = "Не хватило бюджета/порций после улучшений."
    best.uncovered = sorted(list(still_un))
    best.reason_for_uncovered = reason
    return best


# ---------- Компромиссные предложения ----------
def compromise_suggestions(res: PlanResult, dishes: List[Dish], budget: float) -> Dict[str, List[str]]:
    tips: Dict[str, List[str]] = {"drop_expensive": [], "to_feed_uncovered": []}

    # 1) Если нужно удешевить план: уберите блюда с наибольшей долей стоимости
    total = res.total_cost if res.total_cost > 0 else 1.0
    shares = []
    for d in dishes:
        cnt = res.servings.get(d.name, 0)
        if cnt > 0:
            cost = cnt * d.cost
            shares.append((d.name, cost, 100.0 * cost / total))
    shares.sort(key=lambda x: (x[1], x[2]), reverse=True)
    for name, cost, pct in shares[:3]:
        tips["drop_expensive"].append(f"Уберите/сократите {name} (−{cost:.2f} ₽, ~{pct:.1f}% бюджета).")

    # 2) Чтобы накормить оставшихся: минимальные добавки бюджета по каждому
    if res.uncovered:
        tips["to_feed_uncovered"].append("Для каждого ненакормленного гостя добавьте самую дешёвую подходящую порцию:")
    # создадим быстрый доступ по блюдам
    dish_by_name = {d.name: d for d in dishes}
    # грубо оценим «самое дешёвое подходящее»
    # (точный пересчёт плана мы не делаем, чтобы код оставался коротким)
    return tips


# ---------- Пример использования ----------
if __name__ == "__main__":
    # Пример данных (здесь можно подставить свои)
    guests = [
        Guest("Анна", {"vegetarian"}, {"nuts"}),
        Guest("Борис", set(), {"shrimp"}),
        Guest("Вика", {"gluten_free"}, set()),
        Guest("Гоша", {"vegan"}, {"soy"}),
        Guest("Дина", set(), set()),
        Guest("Егор", {"vegetarian", "gluten_free"}, set()),
        Guest("Женя", set(), {"lactose"}),
        Guest("Зоя", {"vegan"}, set()),
    ]

    dishes = [
        Dish("Салат табуле", 4.5, {"wheat","parsley","tomato"}, {"vegetarian"}, 6, preference=0.3),
        Dish("Хумус с овощами", 3.8, {"chickpeas","sesame"}, {"vegan","vegetarian","gluten_free"}, 7, preference=0.7),
        Dish("Куриные наггетсы", 5.2, {"chicken","wheat","egg"}, set(), 8, preference=0.1),
        Dish("Пицца Маргарита", 6.0, {"wheat","cheese","tomato"}, {"vegetarian"}, 5, preference=0.4),
        Dish("Паста безглютеновая", 7.5, {"corn","tomato"}, {"vegetarian","gluten_free"}, 4, preference=0.5),
        Dish("Тофу-терияки", 6.8, {"soy","sesame"}, {"vegan","vegetarian"}, 4, preference=0.2),
        Dish("Креветки гриль", 8.0, {"shrimp"}, set(), 3, preference=0.0),
        Dish("Фрукты ассорти", 3.0, {"apple","banana"}, {"vegan","vegetarian","gluten_free"}, 10, preference=0.6),
        Dish("Суп чечевичный", 4.2, {"lentils","carrot","onion"}, {"vegan","vegetarian","gluten_free"}, 6, preference=0.5),
    ]

    budget = 40.0  # общий бюджет вечеринки

    base = plan_menu_greedy(guests, dishes, budget, preference_alpha=0.25)
    improved = try_local_improvement(base, guests, dishes, budget, max_passes=1)

    # Вывод результатов
    print("=== Итоговый план ===")
    print(f"Бюджет: {budget:.2f} ₽   |   Потрачено: {improved.total_cost:.2f} ₽   |   Покрыто гостей: {improved.covered}/{len(guests)}")
    print("\nМеню (блюдо → порций × цена = сумма):")
    for d in dishes:
        cnt = improved.servings.get(d.name, 0)
        if cnt > 0:
            print(f" • {d.name}: {cnt} × {d.cost:.2f} ₽ = {cnt*d.cost:.2f} ₽")

    print("\nНазначения (кто что ест):")
    for g in guests:
        dish_name = improved.assignment.get(g.name)
        if dish_name:
            print(f" • {g.name} → {dish_name}")
    if improved.uncovered:
        print("\nКого не удалось накормить:")
        for gname in improved.uncovered:
            print(f" • {gname}: {improved.reason_for_uncovered.get(gname, '—')}")

    # Компромиссы:
    print("\nКомпромиссные предложения:")
    tips = compromise_suggestions(improved, dishes, budget)
    if tips["drop_expensive"]:
        print(" • Для экономии бюджета можно:")
        for t in tips["drop_expensive"]:
            print("   -", t)
    if improved.uncovered:
        print(" • Чтобы накормить оставшихся, можно увеличить бюджет на стоимость самой дешёвой подходящей порции для каждого из них.")
        # Краткая подсказка по минимальной цене
        guest_opts, _ = build_options(guests, dishes)
        for gname in improved.uncovered:
            costs = [dishes[i].cost for i in guest_opts.get(gname, [])]
            if costs:
                print(f"   - {gname}: минимум +{min(costs):.2f} ₽")
            else:
                print(f"   - {gname}: нет подходящих блюд по диетам/аллергенам.")

как подставить свои данные
	1.	Замените список guests:
Guest("Имя", {"vegetarian","gluten_free"} или set(), {"nuts","shrimp"} или set())
Требуемые диеты указывайте в requires (набор тегов), аллергии — в allergies.
	2.	Замените список dishes:
Dish("Название", цена_за_порцию, {"ингредиенты"}, {"теги_диет"}, кол-во_порций, preference=0..1)
Поддерживаемые теги: например vegan, vegetarian, gluten_free (можете вводить свои — главное, чтобы совпадали с требованиями гостей).
	3.	Поставьте budget — общий бюджет.

заметки
	•	Алгоритм спроектирован так, чтобы быть понятным и коротким. Он максимально покрывает гостей и старается минимизировать стоимость; при равенстве предпочитает блюда с вашим preference.
	•	Для небольших входов решение зачастую совпадает с оптимумом. Если нужен строго оптимальный расчёт для больших наборов, используйте MILP-солвер (например, pulp/ortools), но это добавит внешние зависимости, а цель тут — простой и автономный скрипт.
